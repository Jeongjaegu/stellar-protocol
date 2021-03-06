## Preamble

```
SEP: <to be assigned>
Title: Dynamic Asset Metadata
Author: OrbitLens <orbit.lens@gmail.com>, Paul Tiplady <paul@qwil.com>
Status: Draft
Created: 2018-09-30
Updated: 2018-09-30
Version 0.1.0
```

## Simple Summary

This SEP extends [SEP-0001](../ecosystem/sep-0001.md) and adds the support of the dynamic asset metadata resolution. It describes a standard way to query asset metadata, thereby allowing the issuer to deal with an unlimited number of assets without defining each of them in `stellar.toml` file.

## Motivation

Current [SEP-0001](../ecosystem/sep-0001.md) specification works excellent for issuers that manage only a few assets. However, `stellar.toml` has a size limit of 100 KB, effectively reducing the maximum possible number of assets to ~300 per file. At present time there is no standard way for the anchor to provide metadata for a large number of assets issued by the same account. There are plenty of use-cases that require thousands of separate assets: bonds, securities, futures, non-fungible tokens. 

## Specification

To allow dynamic asset properties resolution, the issuer implements described REST API endpoints and advertises the existence of a Dynamic Asset Metadata Service through the `stellar.toml` file. Top-level parameter `ASSET_METADATA_SERVER` should contain a fully-qualified URL of a metadata resolution service.

Example of `stellar.toml`:
```
ASSET_METADATA_SERVER="https://anchor.com/assets"
...
```

### Asset Metadata Resolution Flow

- The client discovers the `home_domain` for the asset issuing account and downloads `stellar.toml`.
- Once the file is downloaded and parsed, the client searches for the asset by its code in `[[CURRENCIES]]` section.
- If the metadata for the asset was not found in `[[CURRENCIES]]` section, the client checks for the top-level `ASSET_METADATA_SERVER` parameter.
- If present, the client pulls a metadata from the remote server using the URL defined in `ASSET_METADATA_SERVER` parameter.

### API Endpoints

Dynamic Asset Metadata Service exposes two REST API endpoints.

#### Resolve Asset Metadata Endpoint

`GET <ASSET_METADATA_SERVER>/<ASSET_CODE>`

This API endpoint returns the metadata for a particular asset. The result of the invocation should contain asset metadata following the parameter naming convention described in [SEP-0001](../ecosystem/sep-0001.md) for the `[[CURRENCIES]]` section. 

**Request**

Request Parameters:

- **asset_code** - The code of an asset to look for.

Example: `GET https://anchor.com/assets/AED`

**Response**

On success, the endpoint should return 200 OK HTTP status code and TOML/JSON-encoded object containing asset metadata.
Depending on the value of the ["Accept"](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.1) request header the endpoint should return the response in JSON format for `"application/json"` value or TOML format for `"application/toml"` and `"*/*"` wildcard (default).

Example:
```
code="AED"
name="United Arab Emirates Dirham"
issuer="GAOO3LWBC4XF6VWRP5ESJ6IBHAISVJMSBTALHOQM2EZG7Q477UWC6L7U"
display_decimals=2
```

#### List All Issued Assets Endpoint

`GET <ASSET_METADATA_SERVER>`

This API endpoint returns the list with metadata for all assets issued by the account. It follows Horizon REST API format convention. 

**Request**

Request Parameters:

- **cursor** - Asset code from which to continue the search (referred also as `paging_token` in a result set).
- **order** - Results ordering - `"asc"`(default) or `"desc"`.
- **limit** - Data page size. Default `20`, maximum `200`.

Example: `GET https://anchor.com/assets/?cursor=BOB&order=asc&limit=2`

**Response**

On success, the endpoint should return 200 OK HTTP status code and JSON-encoded object containing the list of the issued assets' metadata. A response result should contain records and navigation links following Horizon response convention.

Example: 

```
{
  "_links": {
    "self": {
      "href": "https://anchor.com/assets/?cursor=BOB&order=asc&limit=2"
    },
    "prev": {
      "href": "https://anchor.com/assets/?cursor=BRL&order=desc&limit=2"
    },
    "next": {
      "href": "https://anchor.com/assets/?cursor=BSD&order=asc&limit=2"
    }
  },
  "_embedded": {
    "records": [
      {
        "code": "BRL",
        "issuer": "GAOO3LWBC4XF6VWRP5ESJ6IBHAISVJMSBTALHOQM2EZG7Q477UWC6L7U"
        "name": "Brazil Real",
        "paging_token": "BRL"
      },
      {
        "code": "BSD",
        "issuer": "GAOO3LWBC4XF6VWRP5ESJ6IBHAISVJMSBTALHOQM2EZG7Q477UWC6L7U"
        "name": "Bahamas Dollar",
        "paging_token": "BSD"
      }
    ]
  }
}

```

####  CORS headers

In order to comply with browser cross-origin access policies, the service should provide wildcard CORS response HTTP header. The following HTTP header must be set for both API endpoints:

```
Access-Control-Allow-Origin: *
```

## Rationale

The "asset metadata" endpoint should support both TOML and JSON format to simplify its usage with different client types. 
Response in TOML format by default is intended to allow using the same parsing logic for both `stellar.toml` and API response.

The "list all" endpoint provides a convenient interface for exchanges, ledger explorers, infrastructure apps, and external services.


## Discussion

- Should we enable multiple issuer accounts? It somewhat complicates the API (`GET https://anchor.com/assets/AED-GAOO3LWBC4XF6VWRP5ESJ6IBHAISVJMSBTALHOQM2EZG7Q477UWC6L7U`) but allows usage of the same server for all assets regardless of the issuer.
- Supporting both TOML and JSON format looks overcomplicated. Nevertheless using only TOML makes integration with external services more complicated (JSON is a clear winner here).   