Requirements & Helper Functions
-------------------------------

### Schema Registration

Schema are registered to an organization. To register a schema, you must
have the appropriate permissions on the organization ACL.

### Running This Code

This requires `reticulate` to be installed and able to import the
Synapse python client. If you do not have `reticulate` set up, then the
following snippet *usually* works.

    install.packages('reticulate')
    reticulate::install_miniconda()
    reticulate::py_install(c("numpy", "pandas", "synapseclient"), pip = TRUE)

The following packages are used. Please install them, if needed.

    install.packages(c("purrr", "glue", "reticulate", "here"))

Import libraries.

    library("purrr")
    library("glue")
    library("reticulate")
    library("here")

These functions are for assisting with registration and access of the
schema. Note that most of these functions are in the synapseAnnotations
repository, but adapted for use with `reticulate`.

    # Functions to register JSON schema file ---------------------------------------

    #' Create schema request body
    #'
    #' @param file Path to JSON Schema file
    #' @param dryRun Should the dryRun field be set to `true`? Defaults to `FALSE`.
    #' @return String containing the JSON request body
    create_body <- function(file, dryRun = FALSE) {
      stopifnot(dryRun %in% c(TRUE, FALSE))
      paste(
        '{"concreteType": "org.sagebionetworks.repo.model.schema.CreateSchemaRequest", "schema": ',
        paste(readLines(file), collapse = ""),
        ', "dryRun": ',
        ifelse(dryRun, 'true', 'false'),
        '}'
      )
    }

    #' Register schema to Synapse
    #'
    #' Given a path to a file containing a JSON schema, registers that schema on
    #' Synapse.
    #'
    #' @param syn Synapse object
    #' @param file Path to JSON Schema file
    #' @param dryRun Should the dryRun field be set to `true`? Defaults to `FALSE`.
    #' @return Token object containing the async token used to monitor the job
    register_schema <- function(syn, file, dryRun = FALSE) {
      syn$restPOST(
        uri = "/schema/type/create/async/start",
        body = create_body(file, dryRun = dryRun)
      )
    }

    #' Get status of registered schema
    #'
    #' @param syn Synapse object
    #' @param token Async token
    #' @return Response object containing schema
    get_registration_status <- function(syn, token) {
      processing <- TRUE
      ## Check results of registering schema, retrying if the async job hasn't
      ## completed yet
      while (processing) {
        result <- syn$restGET(
          uri = glue("/schema/type/create/async/get/{token}")
        )
        ## synapser doesn't return the status codes unfortunately, so we check the
        ## response object to determine what to do. If it contains "jobState",
        ## that's part of the AsynchronousJobStatus, i.e. the async job hasn't
        ## completed yet.
        if (!"jobState" %in% names(result)) {
          processing <- FALSE
        }
      }
      result
    }

    #' Register schema and report on its status
    #'
    #' Registers a schema and then monitors the asynchronous job until it completes.
    #'
    #' @param syn Synapse object
    #' @param file Path to JSON Schema file
    #' @param dryRun Should the dryRun field be set to `true`? Defaults to `FALSE`.
    #' @return `TRUE` if schema was successfully registered; `FALSE` otherwise.
    register_and_report <- function(syn, file, dryRun = FALSE) {
      message(glue("Registering {file} with dryRun = {dryRun}"))
      ## register_schema() can fail if e.g. schemas contain non-ASCII characters --
      ## in this case we catch that error and provide the message.
      token <- try(register_schema(syn, file, dryRun = dryRun), silent = TRUE)
      if (inherits(token, "try-error")) {
        message(token)
        return(FALSE)
      }
      ## Note that this is written for how synapser works.
      ## It should be updated for reticulate, but it currently works as is.
      ## Reasoning for synapser below:
      ##   If we get a 400 status code, synapser will throw an error -- we want to
      ##   catch that and provide the message but not raise an actual R error.
      status <- try(get_registration_status(syn, token$token), silent = TRUE)
      if (inherits(status, "try-error")) {
        message(status)
        FALSE
      } else {
        TRUE
      }
    }

Usage
-----

Import the `synapseclient` and log into Synapse.

    ## Log in to synapse
    synapse <- import("synapseclient")
    syn <- synapse$Synapse()
    syn$login()

### Registering Schema

First find the schema files.

    files <- list.files(
      here("schema"),
      pattern = ".json$",
      full.names = TRUE
    )

For schemas that use referencing, there is an order to how schemas are
registered. All referenced schema must be registered before they can be
referenced. For the schema in this project, I’ve numbered the schema to
indicate at what ‘level’ they are: a higher number means the sooner it
must be registered. For example, the main schema is at level 01 and
references both schema at level 02. The schema at level 02 must be
registered before the main schema, at level 01, can be registered. The
nice thing about labeling the level at the front of the file name is
that the files are already ordered from level 01 to 05. To register
these in the correct order, all that’s needed is to reverse the order.

    files <- rev(files)

Some of these might already be registered. If these were versioned, it
would be impossible to register them a second time. In the case of
versioned schema, do a dry-run first to check which ones do not need to
be registered. The following example snippet will return a boolean list
that can be used to filter out the schemas that already registered in
Synapse (i.e. boolean value is `FALSE`).

**Note:** Any `FALSE` returns from the dry-run could be from the schema
already existing in Synapse or from the schema being altogether invalid.
Here, we are assuming the schemas are valid and that any `FALSE` is
purely due to not being registered, yet. Again, if a schema is not
versioned and the file is valid, this will return `TRUE`, even if the
schema is identical to latest registered version.

    should_register <- map_lgl(files, register_and_report, syn = syn, dryRun = TRUE)

Check to see if any need registering and, if so, filter the list of
files.

    if(any(should_register)) {
      ## Only keep `TRUE` files
      files <- files[should_register]
    } else {
      files <- NA
    }

Register files by looping through all the files that should be
registered. Output will be `TRUE` for each file that was registered.

    if(any(should_register)) {
      ## Register files
      map_lgl(files, register_and_report, syn = syn) 
    }

### Binding Schema

Any schema that is registered in Synapse can be bound to a project,
directory, or other entity in Synapse.

    bind_schema <- function(syn, entity_id, schema_id) {
      ## Build strings outside of rest call for reasons
      uri <- glue::glue("/entity/{entity_id}/schema/binding")
      body <- glue::glue("{{entityId: \"{entity_id}\", schema$id: \"{schema_id}\"}}")
      syn$restPUT(
        uri = uri,
        body = body
      )
    }

Use helper function above to bind a schema to an entity. In this
example, I binding the nkauer-ad.main schema to the Studies directory in
my test project.

    desired_entity_id <- "syn25421165"
    desired_schema_id <- "nkauer-ad.main"

    bind_schema(syn, desired_entity_id, desired_schema_id)

### Get Validation Schema

Synapse needs to create a validation schema based on the registered
schema. This is especially important for schema with references to other
schema; Synapse needs to follow the references and gather the needed
information. You can pull down the validation schema that Synapse
creates.

    get_validation_schema <- function(syn, schema_id) {
      ## Start the job to compile the validation schema
      token <- syn$restPOST(
        uri = "/schema/type/validation/async/start",
        body = glue::glue("{{$id: \"{schema_id}\"}}")
      )
      ## Should wait until finished
      schema <- get_schema_status(syn, token)
      schema
    }
    ## Have to keep checking on status
    get_schema_status <- function(syn, token) {
      processing <- TRUE
      ## Check results of registering schema, retrying if the async job hasn't
      ## completed yet
      while (processing) {
        result <- syn$restGET(
          uri = glue::glue("/schema/type/validation/async/get/{token}")
        )
        ## synapser doesn't return the status codes unfortunately, so we check the
        ## response object to determine what to do. If it contains "jobState",
        ## that's part of the AsynchronousJobStatus, i.e. the async job hasn't
        ## completed yet.
        if (!"jobState" %in% names(result)) {
          processing <- FALSE
        }
      }
      result
    }

Here is an example that grabs the nkauer-ad.main validation schema.

    desired_schema_id <- "nkauer-ad.main"
    validation_schema <- get_validation_schema(syn, desired_schema_id)

This is a large schema. Can turn it into prettified json with jsonlite.

    json_string <- jsonlite::prettify(jsonlite::toJSON(validation_schema))

    ## Recommend writing to a file versus looking at the string in R
    write(json_string, file = "nkauer-ad.main_dereferenced.json")

### Validate Annotations Using Schema

The title of this section is a trick because Synapse is already working
on validating every file against the schema. You don’t have to do
anything other than wait patiently.

### Get Validation Results

Grabbing the validation results is simple.

    get_validation_results <- function(syn, entity_id) {
      syn$restGET(
        uri = glue::glue("/entity/{entity_id}/schema/validation")
      )
    }

    ## Check if results say the file is valid or invalid
    ## Overly simplistic function...
    is_valid_result <- function(validation_result) {
      validation_result$isValid
    }

There is a table with all files that are in my ‘Studies’ directory,
which are bound to the nkauer-ad.main schema. Loop through all of them
to get their annotation validation status.

    ## Grab all synIDs that have a schema bound
    ## I have these in a fileview for ease
    fileview <- "syn25421275"
    all_files <- syn$tableQuery(
      glue::glue("SELECT id FROM {fileview}")
    )$asDataFrame()$id

    all_results <- purrr::map(all_files, ~ get_validation_results(syn, .))
    is_valid <- purrr::map_lgl(all_results, ~ is_valid_result(.))

    ## Check if all or any are valid
    any(is_valid)

Parsing the returned validation results for invalid annotations is left
as an exercise…

### Validation Examples

Below are some validation examples, but first is a function to simplify
printing the results to a file.

    #' Output json validation results to file
    #' @param path File location to write to
    #' @param results Results from Synapse validation
    output_validation_file <- function(path, results) {
      json_string <- jsonlite::prettify(jsonlite::toJSON(results))
      write(json_string, file = path)
    }

#### No schema bound

This file has no schema associated with it. Asking for validation
results causes an error.

    ## Example 1
    no_schema_results <- tryCatch(
      {
        get_validation_results(syn, "syn25421195") 
      },
      error = function(e) {
        print(e$message)
      }
    )

    ## [1] "SynapseHTTPError: 404 Client Error: \nValidationResults do not exist for objectId: 'syn25421195' and objectType: 'entity'\n\nDetailed traceback:\n  File \"/home/rstudio/.virtualenvs/r-reticulate/lib/python3.8/site-packages/synapseclient/client.py\", line 3827, in restGET\n    response = self._rest_call('get', uri, None, endpoint, headers, retryPolicy, requests_session, **kwargs)\n  File \"/home/rstudio/.virtualenvs/r-reticulate/lib/python3.8/site-packages/synapseclient/client.py\", line 3811, in _rest_call\n    self._handle_synapse_http_error(response)\n  File \"/home/rstudio/.virtualenvs/r-reticulate/lib/python3.8/site-packages/synapseclient/client.py\", line 3782, in _handle_synapse_http_error\n    exceptions._raise_for_status(response, verbose=self.debug)\n  File \"/home/rstudio/.virtualenvs/r-reticulate/lib/python3.8/site-packages/synapseclient/core/exceptions.py\", line 160, in _raise_for_status\n    raise SynapseHTTPError(message, response=response)\n"

#### Valid vs Invalid with Enumerated Values

The following two files are associated with the large AD test schema,
but have short paths, making them relatively simple. They only have 4
required annotations, *consortium*, *study*, *resourceType*, and
*fileFormat*. Their path is guided purely by *resourceType* value of
*tool*. However, this looks complex when a single enumerated value is
wrong. In the invalid case, the value *neato* for *fileFormat* is not
acceptable.

    ## Example 2
    valid_results <- get_validation_results(syn, "syn25421327")
    output_validation_file(
      here("validation_results", "example2_valid.json"),
      valid_results
    )

    invalid_results <- get_validation_results(syn, "syn25532081")
    output_validation_file(
      here("validation_results", "example2_invalid.json"),
      invalid_results
    )

#### Simple Invalid Case

The following example has a simple schema that only requires *waterpH*,
which has an allowed range of 0-14. The annotation is set to 42.

    ## Example 3
    invalid_results <- get_validation_results(syn, "syn25486167")
    output_validation_file(
      here("validation_results", "example3_invalid.json"),
      invalid_results
    )

#### Slightly Complex Invalid Case – Due to Bad Schema

The following file is associated with the large test AD schema. This
file still has a relatively short path based on the *resourceType* value
of *metadata*. The annotations are all correct, but the logic is not
quite right for working with schema. In this case, it thinks
*isMultiSpecimen* should be an object and it doesn’t like how I’m trying
to check if *isCellLine* is *false*. While this is a bad schema design,
the complexity of validation results is still relevant. This is only for
two problems in a relatively short path.

    ## Example 4
    invalid_results <- get_validation_results(syn, "syn25421297")
    output_validation_file(
      here("validation_results", "example4_invalid.json"),
      invalid_results
    )
