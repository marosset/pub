# Pub: cmd line for the Azure Cloud Partner Portal

[![Actions Status](https://github.com/devigned/pub/workflows/ci/badge.svg)](https://github.com/devigned/pub/actions)

Pub enables command line automation for working with offers on
[Azure Marketplace](https://azuremarketplace.microsoft.com/) and on
[AppSource](https://appsource.microsoft.com/). For more information on the Cloud Partner
Portal, check out [Azure Docs](https://docs.microsoft.com/en-us/azure/marketplace/cloud-partner-portal-orig/cloud-partner-portal-getting-started-with-the-cloud-partner-portal).

The command line interface for `pub` is simply a facade over the [REST API exposed by the
Cloud Partner Portal](https://docs.microsoft.com/en-us/azure/marketplace/cloud-partner-portal-orig/cloud-partner-portal-api-overview).

## Install

Easiest way is to use golang, but there are also precompiled binaries on the [releases page](https://github.com/devigned/pub/releases/).

```bash
$ go get github.com/devigned/pub
...
$ pub -h
...
```

## Usage

From the top level, `pub` offers access to view and manipulate resources.

```bash
pub provides a command line interface for the Azure Cloud Partner Portal

Usage:
  pub [command]

Available Commands:
  help        Help about any command
  offers      a group of actions for working with offers
  operations  a group of actions for working with offer operations
  publishers  a group of actions for working with publishers
  skus        a group of actions for working with SKUs
  version     Print the git ref
  versions    a group of actions for working with versions

Flags:
  -v, --api-version string   the API version override (default "2017-10-31")
      --config string        config file (default is $HOME/.pub.yaml)
  -h, --help                 help for pub

Use "pub [command] --help" for more information about a command.
```

### Object Model

- Publishers have many offers
- Offers can have many SKUS
- Offers can have many operations with only one active operation
- SKUs can have many versions

### Authentication

There are two ways to authenticate to `pub`. Both paths use Azure Active Directory (AAD) to
authenticate against the `https://cloudpartner.azure.com` resource URI. Using a token
generated by [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
is probably the easiest way to get started. When you are ready to fully automate the
publication process, using a AAD Service Principal / Application will make it easier to
inject environment variables into a build process.

#### Using an Azure Active Directory Token via Azure CLI

With the commands below, you can fetch an AAD token via Azure CLI and use it to auth calls
in pub.

```bash
token=$(az account get-access-token --resource https://cloudpartner.azure.com --query "accessToken" -o tsv)
AZURE_TOKEN=$token
pub publishers list
```

#### Using an AAD Application and Service Principal

**Prerequisite:** [setup an AAD Application and Service Principal](https://docs.microsoft.com/en-us/azure/marketplace/cloud-partner-portal-orig/cloud-partner-portal-api-prerequisites#create-a-service-principal-in-your-azure-active-directory-tenant).

After setting up the AAD App / Service Principal, set the following environment variables:

```bash
export AZURE_TENANT_ID=<tenant uuid>
export AZURE_CLIENT_ID=<application / client uuid>
export AZURE_CLIENT_SECRET=<service principal secret / password>
pub publishers list
```

### Command Output

All command output and any complex input is formatted as JSON. Definitely recommend using
[jq](https://stedolan.github.io/jq/) or something similar.

### Publishers

Provides the ability to query the publishers.

```bash
$ ./bin/pub publishers -h
a group of actions for working with publishers

Usage:
  pub publishers [command]

Available Commands:
  list        list all publishers

Flags:
  -h, --help   help for publishers

Global Flags:
  -v, --api-version string   the API version override (default "2017-10-31")
      --config string        config file (default is $HOME/.pub.yaml)

Use "pub publishers [command] --help" for more information about a command.

$ ./bin/pub publishers list | jq
[
  {
    "id": "your-publisher-id",
    "definition": {
      "displayText": "Your Publisher Name",
      "offerTypeCategories": [
        "someCategories"
      ],
      "sellerId": 1234567
    }
  }
]
```

### Offers

Offers is the heart of most of the Cloud Partner Portal workflow. For more details, see [the
marketplace offers documentation](https://docs.microsoft.com/en-us/azure/marketplace/cloud-partner-portal/cpp-marketplace-offers).
Offers represent a marketing / description header over a set of `SKUs`.

To take a publication to market, run the following workflow:

`put offer -> put version -> publish offer -> test offer -> live`

The publish and live commands both start a long running operation which can be observed via
the `operations` commands. There can only be 1 operation running on an offer at a time.

```bash
$ pub offers
a group of actions for working with offers

Usage:
  pub offers [command]

Available Commands:
  list        list all offers
  live        go live with an offer (make available to the world)
  publish     publish an offer
  put         create or update an offer
  show        show an offer
  status      show status for an offer
...
```

### SKUs

A `SKU`, or a `Plan` in the REST API, contains details for a specific type of offering. For example,
the `SKU` represents a VM Image or other specialized, versioned type under an offering.

```bash
$ pub skus
a group of actions for working with SKUs

Usage:
  pub skus [command]

Available Commands:
  list        list all SKUs for a given offer and publisher
...
```

### Versions

Versions are the lowest level resource in the marketplace. For example, in a VM Image, this would
consist of the URI to the VHD media and details related to that specific version of the image.

```bash
$ pub versions
a group of actions for working with versions

Usage:
  pub versions [command]

Available Commands:
  list        list all versions for a given plan
  put         put a version for a given plan
  show        show a version for a given plan
...
```

### Operations

Operations provide insight into the workflow and status of the publication process.

```bash
$ pub operations
a group of actions for working with offer operations

Usage:
  pub operations [command]

Available Commands:
  cancel      cancel the active operation for a given offer and print the operations
  list        list operations and optionally filter by status
  show        show an operation by Id
...
```

### Debug Output

If you want to see more details about the HTTP requests being made, run any command with
`DEBUG=true`.

```bash
$ DEBUG=true pub publishers list
GET /api/publishers?api-version=2017-10-31 HTTP/1.1
Host: cloudpartner.azure.com

HTTP/1.1 200 OK
Transfer-Encoding: chunked
Cache-Control: no-cache
Content-Type: application/json; charset=utf-8
Date: Sun, 03 Nov 2019 16:55:07 GMT
Expires: -1
Pragma: no-cache
Server: Microsoft-IIS/10.0
Vary: Accept-Encoding
X-Aspnet-Version: 4.0.30319
X-Content-Type-Options: nosniff
X-Ms-Correlation-Request-Id: dc12d384-6c53-4880-9b84-de1e89cbf9c7
X-Powered-By: ASP.NET

8a
[{"version":0,"id":"your-publisher-id","definition":{"displayText":"Your publisher name,"offerTypeCategories":["someCategory"],"sellerId":1234456}}]
0

[{"id":"your-publisher-id","definition":{"displayText":"Your publisher name,"offerTypeCategories":["someCategory"],"sellerId":1234456}}]
```
