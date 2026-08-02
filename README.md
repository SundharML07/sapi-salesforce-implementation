# sapi-salesforce-implementation

Mule 4 implementation project for the `sapi-salesforce` API. It exposes basic Salesforce operations through APIKit HTTP routes and MCP tools.

## What It Does

- Queries Salesforce records using SOQL.
- Creates Salesforce records for a provided SObject type.
- Updates or upserts Salesforce records.
- Exposes the same operations as MCP tools through the Mule MCP connector.
- Uses API Manager autodiscovery for the deployed API instance.

## Tech Stack

- Mule runtime `4.11.0`
- Java `17`
- Maven
- Mule Maven Plugin `4.8.0`
- MUnit `3.7.0`
- Salesforce Connector `11.3.3`
- MCP Connector `1.3.1`

## Main Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/query?soql=...` | Run a Salesforce SOQL query. |
| `POST` | `/create?objectname=...` | Create a Salesforce record. |
| `POST` | `/update?objectname=...` | Update a Salesforce record. |
| `POST` | `/update?objectname=...&operation=upsert&externalidfieldname=...` | Upsert a Salesforce record. |
| `GET` | `/console/*` | APIKit console. |

## Project Structure

```text
src/main/mule/
  sapi-salesforce.xml          APIKit listener and route mapping
  global.xml                   global config, Salesforce config, secure properties
  error.xml                    global error handling
  mcp-tools.xml                MCP tool wrappers
  implementation/
    query.xml                  Salesforce query flow
    create.xml                 Salesforce create flow
    update.xml                 Salesforce update and upsert flow

src/main/resources/properties/
  config-global.yaml           API asset and API Manager instance config
  config-local.yaml            local encrypted Salesforce and HTTP config
  config-dev.yaml              dev encrypted Salesforce and HTTP config

src/test/munit/                MUnit test suites
exchange-docs/                 Anypoint Exchange documentation
```

## Configuration

The app loads environment-specific config from:

```text
src/main/resources/properties/config-${env}.yaml
```

Current `global.xml` sets `env` to `local`, so local runs use `config-local.yaml` unless that is changed.

Required secure properties:

- `salesforce.client_id`
- `salesforce.client_secret`
- `salesforce.token_url`

Required runtime environment variable:

- `MULE_SECURE_KEY`

## Deployment

The Maven configuration supports CloudHub 2.0 deployment through a MuleSoft Connected App. Important deployment defaults are defined in `pom.xml`:

- CloudHub target: `Cloudhub-US-East-2`
- App name: `sapi-salesforce-implementation`
- Environment property: `env=dev`
- Commit traceability property: `commit.shortSha`

Deployment setup and required GitHub secrets are documented in [CI_CD_SETUP.md](CI_CD_SETUP.md).
