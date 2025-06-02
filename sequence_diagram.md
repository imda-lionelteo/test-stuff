# Moonshot Project - Sequence Diagrams

## Overview
This document contains sequence diagrams for the core flows in the Moonshot project, illustrating how different components interact during key operations.

## 1. Benchmarking Flow

### Runner Creation and Recipe Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant API as APIRunner
    participant Runner
    participant Storage
    participant DB as DBInterface
    participant Recipe
    participant Dataset
    participant Connector
    participant Metric
    
    %% Runner Creation
    User->>API: api_create_runner(name, endpoints)
    API->>Runner: create(runner_args)
    Runner->>Storage: create_object(RUNNERS, runner_id, info)
    Runner->>Storage: create_database_connection(DATABASES, runner_id)
    Storage->>DB: create connection
    DB-->>Storage: database_instance
    Storage-->>Runner: database_instance
    Runner->>Storage: create_database_table(cache_table)
    Storage->>DB: execute CREATE TABLE
    Runner-->>API: runner_instance
    API-->>User: runner_id
    
    %% Recipe Execution
    User->>API: run_recipes(runner_id, recipes)
    API->>Runner: load(runner_id)
    Runner->>Storage: read_object(RUNNERS, runner_id)
    Storage-->>Runner: runner_data
    
    loop For each recipe
        Runner->>Recipe: read(recipe_id)
        Recipe->>Storage: read_object(RECIPES, recipe_id)
        Storage-->>Recipe: recipe_data
        Recipe-->>Runner: recipe_args
        
        loop For each dataset in recipe
            Runner->>Dataset: read(dataset_id)
            Dataset->>Storage: read_object_with_iterator(DATASETS, dataset_id)
            Storage-->>Dataset: dataset_data
            Dataset-->>Runner: dataset_iterator
            
            loop For each prompt in dataset
                Runner->>Connector: get_prediction(prompt_args, connector)
                Connector->>Connector: rate_limited_call()
                Connector->>Connector: perform_retry()
                Note over Connector: LLM API Call with<br/>rate limiting & retries
                Connector-->>Runner: prediction_result
                
                Runner->>Storage: create_database_record(cache, result)
                Storage->>DB: INSERT result
            end
        end
        
        Runner->>Metric: get_results(prompts, predictions, targets)
        Metric-->>Runner: evaluation_scores
        
        Runner->>Storage: create_object(RESULTS, result_id, results)
        Storage-->>Runner: result_file_path
    end
    
    Runner-->>API: execution_complete
    API-->>User: results_summary
```

## 2. Red-Teaming Flow

### Session Creation and Interactive Chat Flow

```mermaid
sequenceDiagram
    participant User
    participant API as APISession
    participant Runner
    participant Session
    participant SessionMetadata
    participant Chat
    participant Connector
    participant AttackModule
    participant Storage
    participant DB as DBInterface
    
    %% Session Creation
    User->>API: api_create_session(runner_id, endpoints, args)
    API->>Runner: load(runner_id)
    Runner->>Storage: read_object(RUNNERS, runner_id)
    Storage-->>Runner: runner_data
    
    API->>Session: __init__(runner_id, type, args, db, endpoints)
    Session->>Storage: create_database_table(session_metadata_table)
    Session->>Storage: create_database_table(chat_history_table)
    
    Session->>SessionMetadata: __init__(session_data)
    Session->>Storage: create_database_record(session_metadata)
    Storage->>DB: INSERT session metadata
    
    Session-->>API: session_instance
    API-->>User: session_created
    
    %% Interactive Chat Flow
    User->>API: send_chat_message(runner_id, prompt)
    API->>Session: load(database_instance)
    Session->>Storage: read_database_record(session_metadata)
    Storage->>DB: SELECT session data
    DB-->>Storage: session_metadata
    Storage-->>Session: session_data
    
    Session->>AttackModule: prepare_attack_prompt(original_prompt)
    AttackModule-->>Session: modified_prompt
    
    Session->>Chat: create_chat_record(prompt_data)
    
    Session->>Connector: get_prediction(modified_prompt)
    Note over Connector: Rate-limited call to LLM
    Connector-->>Session: response
    
    Session->>Chat: update_chat_record(response, duration)
    Session->>Storage: create_database_record(chat_history, chat_data)
    Storage->>DB: INSERT chat record
    
    Session->>Session: update_progress()
    Session-->>API: chat_response
    API-->>User: response_with_metadata
    
    %% Session Management
    User->>API: api_update_context_strategy(runner_id, strategy)
    API->>Session: update_context_strategy(db, runner_id, strategy)
    Session->>Storage: update_database_record(session_metadata)
    Storage->>DB: UPDATE session SET context_strategy
    Session-->>API: update_success
    API-->>User: strategy_updated
```

## 3. Dataset Management Flow

### Dataset Creation and Processing Flow

```mermaid
sequenceDiagram
    participant User
    participant API as APIDataset
    participant Dataset
    participant Storage
    participant HuggingFace as HF
    participant FileSystem as FS
    
    %% CSV Dataset Creation
    User->>API: api_create_dataset(csv_file_path, metadata)
    API->>Dataset: create(dataset_args)
    
    Dataset->>Dataset: convert_data(csv_file_path)
    Dataset->>FS: read CSV file
    FS-->>Dataset: csv_data
    Dataset->>Dataset: validate_headers(input, target)
    Dataset->>Dataset: convert_to_iterator()
    
    Dataset->>Storage: create_object_with_iterator(DATASETS, id, info, iterator)
    Storage->>Storage: write_metadata_to_file()
    
    loop For each data row
        Storage->>Storage: write_example_to_file(example)
    end
    
    Storage-->>Dataset: file_path
    Dataset-->>API: dataset_id
    API-->>User: dataset_created
    
    %% HuggingFace Dataset Download
    User->>API: api_create_dataset(hf_args)
    API->>Dataset: create(dataset_args)
    
    Dataset->>Dataset: download_hf(dataset_name, config, split)
    Dataset->>HF: load_dataset(name, config, split)
    HF-->>Dataset: hf_dataset
    
    loop For each example in HF dataset
        Dataset->>Dataset: transform_example(input_cols, target_col)
    end
    
    Dataset->>Storage: create_object_with_iterator(DATASETS, id, info, iterator)
    Storage-->>Dataset: file_path
    Dataset-->>API: dataset_id
    API-->>User: dataset_created
    
    %% Dataset Reading
    User->>API: api_get_dataset(dataset_id)
    API->>Dataset: read(dataset_id)
    Dataset->>Storage: read_object_with_iterator(DATASETS, dataset_id)
    Storage->>Storage: read_metadata()
    Storage->>Storage: create_iterator(examples)
    Storage-->>Dataset: dataset_data_with_iterator
    Dataset-->>API: dataset_args
    API-->>User: dataset_info
```

## 4. Connector Management Flow

### Connector Endpoint Creation and LLM Interaction Flow

```mermaid
sequenceDiagram
    participant User
    participant API as APIConnectorEndpoint
    participant ConnectorEndpoint
    participant Storage
    participant Connector
    participant LLM as External_LLM
    participant RateLimit as RateLimiter
    
    %% Connector Endpoint Creation
    User->>API: api_create_endpoint(name, connector_type, uri, token, model, params)
    API->>API: create ConnectorEndpointArguments(args)
    API->>ConnectorEndpoint: create(endpoint_args)
    ConnectorEndpoint->>ConnectorEndpoint: generate ep_id = slugify(name)
    ConnectorEndpoint->>Storage: create_object(CONNECTORS_ENDPOINTS, ep_id, config)
    Storage-->>ConnectorEndpoint: success
    ConnectorEndpoint-->>API: endpoint_id
    API-->>User: endpoint_created
    
    %% Connector Loading and Prediction Flow
    User->>API: get_prediction(prompt, endpoint_id)
    API->>ConnectorEndpoint: read(endpoint_id)
    ConnectorEndpoint->>Storage: read_object(CONNECTORS_ENDPOINTS, endpoint_id)
    Storage-->>ConnectorEndpoint: endpoint_config
    ConnectorEndpoint->>ConnectorEndpoint: add creation_datetime
    ConnectorEndpoint-->>API: ConnectorEndpointArguments
    
    API->>Connector: load(endpoint_args)
    Connector->>Connector: get_instance(connector_type, filepath)
    Note over Connector: Dynamic loading of<br/>connector implementation
    Connector->>Connector: __init__(endpoint_args)
    Note over Connector: Initialize rate limiter,<br/>semaphore, token bucket
    
    %% Rate Limited Prediction Call
    API->>Connector: get_prediction(prompt_args, connector)
    Connector->>Connector: @rate_limited wrapper
    Connector->>RateLimit: acquire semaphore()
    RateLimit->>RateLimit: _add_tokens() - refill bucket
    
    alt Tokens >= 1
        RateLimit->>RateLimit: consume 1 token
        RateLimit-->>Connector: proceed
    else Tokens < 1
        RateLimit->>RateLimit: calculate sleep_time = (1-tokens)/rate_limiter
        RateLimit->>RateLimit: await sleep(sleep_time)
        RateLimit->>RateLimit: _add_tokens() - refill after sleep
        RateLimit->>RateLimit: consume 1 token
        RateLimit-->>Connector: proceed
    end
    
    %% Retry Mechanism
    Connector->>Connector: @perform_retry wrapper
    
    loop Retry attempts (max_attempts from config)
        Connector->>LLM: get_response(prompt) - HTTP request
        
        alt Success Response
            LLM-->>Connector: ConnectorResponse(response, None, None)
            Connector->>Connector: log successful prediction
            break
        else API Error (Rate limit, timeout, etc.)
            LLM-->>Connector: Exception/Error
            Connector->>Connector: perform_retry_callback(connector_id, retry_state)
            Connector->>Connector: log retry attempt with error details
            
            alt Final attempt
                Connector->>Connector: raise exception
            else More attempts remaining
                Connector->>Connector: wait_random_exponential(min=1, max=60)
            end
        end
    end
    
    Connector->>Connector: record end_time and calculate duration
    Connector->>RateLimit: release semaphore()
    Connector-->>API: ConnectorPromptArguments(updated with prediction & duration)
    
    opt Prompt Callback exists
        API->>API: prompt_callback(updated_prompt_args, connector_id)
    end
    
    API-->>User: prediction_result
```

## 5. Storage Operations Flow

### File and Database Operations Flow

```mermaid
sequenceDiagram
    participant Component
    participant Storage
    participant IOModule
    participant DBInterface
    participant FileSystem as FS
    participant Database as DB
    
    %% Object Creation with Iterator
    Component->>Storage: create_object_with_iterator(type, id, info, iterator)
    Storage->>Storage: validate_object_type(type)
    Storage->>Storage: get_filepath(type, id, extension)
    
    Storage->>IOModule: get_instance(jsonio)
    IOModule-->>Storage: io_instance
    
    Storage->>IOModule: create_file_with_iterator(filepath, info, keys, iterator)
    IOModule->>FS: create_file(filepath)
    IOModule->>FS: write_metadata(info)
    
    loop For each item in iterator
        IOModule->>FS: write_item(item)
    end
    
    IOModule-->>Storage: success
    Storage-->>Component: filepath
    
    %% Database Operations
    Component->>Storage: create_database_connection(type, id, extension)
    Storage->>Storage: get_filepath(type, id, extension)
    Storage->>DBInterface: get_instance(sqlite, filepath)
    DBInterface->>DB: connect(filepath)
    DB-->>DBInterface: connection
    DBInterface-->>Storage: db_instance
    Storage-->>Component: db_instance
    
    Component->>Storage: create_database_table(db_instance, sql)
    Storage->>DBInterface: execute_query(sql)
    DBInterface->>DB: CREATE TABLE
    DB-->>DBInterface: success
    DBInterface-->>Storage: success
    Storage-->>Component: table_created
    
    Component->>Storage: create_database_record(db_instance, data, sql)
    Storage->>DBInterface: execute_query(sql, data)
    DBInterface->>DB: INSERT INTO table VALUES
    DB-->>DBInterface: record_id
    DBInterface-->>Storage: record_id
    Storage-->>Component: record_created
    
    %% Object Reading with Iterator
    Component->>Storage: read_object_with_iterator(type, id, extension, keys, iterator_keys)
    Storage->>IOModule: get_instance(jsonio)
    IOModule-->>Storage: io_instance
    
    Storage->>IOModule: read_file_iterator(filepath, keys, iterator_keys)
    IOModule->>FS: read_metadata(filepath)
    FS-->>IOModule: metadata
    
    IOModule->>IOModule: create_iterator(iterator_keys)
    IOModule-->>Storage: data_with_iterator
    Storage-->>Component: object_data
```

## 6. Recipe and Cookbook Execution Flow

### Comprehensive Testing Flow

```mermaid
sequenceDiagram
    participant User
    participant Runner
    participant Cookbook
    participant Recipe
    participant Dataset
    participant PromptTemplate
    participant Connector
    participant Metric
    participant Results
    participant Storage
    
    %% Cookbook Execution
    User->>Runner: run_cookbooks(cookbook_ids)
    
    loop For each cookbook
        Runner->>Cookbook: read(cookbook_id)
        Cookbook->>Storage: read_object(COOKBOOKS, cookbook_id)
        Storage-->>Cookbook: cookbook_data
        Cookbook-->>Runner: recipe_list
        
        loop For each recipe in cookbook
            Runner->>Recipe: read(recipe_id)
            Recipe->>Storage: read_object(RECIPES, recipe_id)
            Storage-->>Recipe: recipe_data
            Recipe-->>Runner: recipe_config
            
            %% Dataset Processing
            loop For each dataset in recipe
                Runner->>Dataset: read(dataset_id)
                Dataset-->>Runner: dataset_iterator
                
                %% Prompt Template Application
                alt Prompt template exists
                    Runner->>PromptTemplate: read(template_id)
                    PromptTemplate->>Storage: read_object(PROMPT_TEMPLATES, template_id)
                    Storage-->>PromptTemplate: template_data
                    PromptTemplate-->>Runner: template
                end
                
                %% Prompt Processing
                loop For each prompt in dataset
                    alt With prompt template
                        Runner->>Runner: apply_template(prompt, template)
                    end
                    
                    Runner->>Connector: get_prediction(processed_prompt)
                    Note over Connector: Rate-limited LLM call
                    Connector-->>Runner: prediction
                    
                    Runner->>Storage: cache_result(prompt, prediction, metadata)
                end
            end
            
            %% Metric Evaluation
            loop For each metric in recipe
                Runner->>Metric: get_results(prompts, predictions, targets)
                Metric->>Metric: calculate_scores()
                Metric-->>Runner: metric_scores
            end
            
            %% Results Compilation
            Runner->>Results: compile_recipe_results(scores, metadata)
            Results->>Storage: create_object(RESULTS, result_id, data)
            Storage-->>Results: result_file
            Results-->>Runner: recipe_results
        end
        
        Runner->>Results: compile_cookbook_results(recipe_results)
        Results->>Storage: create_object(RESULTS, cookbook_result_id, data)
        Storage-->>Results: cookbook_result_file
        Results-->>Runner: cookbook_results
    end
    
    Runner-->>User: execution_complete(all_results)
```

## Flow Summary

### Key Interaction Patterns:

1. **Asynchronous Processing**: Most operations support async execution with progress callbacks
2. **Rate Limiting**: Connector interactions implement token bucket rate limiting
3. **Retry Mechanisms**: Robust error handling with exponential backoff
4. **Caching**: Results are cached in databases for performance
5. **Iterator Pattern**: Large datasets are processed using iterators to manage memory
6. **Factory Pattern**: Components are created through factory methods
7. **State Management**: Sessions and runs maintain state through database persistence

### Performance Considerations:

- **Concurrent Execution**: Multiple connectors can run in parallel with semaphore control
- **Memory Efficiency**: Large datasets use iterators instead of loading all data
- **Database Optimization**: Results are cached to avoid redundant LLM calls
- **Progress Tracking**: Long-running operations provide real-time progress updates

These sequence diagrams illustrate the sophisticated orchestration of components in the Moonshot system, showing how it manages complex workflows while maintaining performance, reliability, and extensibility. 
