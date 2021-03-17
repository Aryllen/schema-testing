# schema-testing

This repository is testing out schemas for Synapse annotations.

## Naming convention

Each file is a mini-schema and all combine to create one full annotation schema. The schema must be registered in reverse order with the main schema (01-main.json) being registered last. I have attempted to name these files so that one could start from the last group of files, starting with 05, and work their way up to the main file, backwards.

## Organization

All schema have been registered under the Synapse schema organization 'nkauer'