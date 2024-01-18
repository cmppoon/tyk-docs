# API Templates

## Overview

API Templates is a new feature designed to streamline the process of defining APIs using the OpenAPI Specification (OAS). These templates serve as a foundational or default structure for various segments of an OAS API definition, enhancing the efficiency and consistency of API creation and importation.

## Key Features

- **Starting Point for API Definitions**: API templates provide a base structure for OAS API definitions, ensuring uniformity and saving time in the API creation process.
- **Application during Import or Creation**: Templates can be applied both when importing an existing API or during the creation of a new one.
- **Easy Identification through IDs**: Templates are identifiable either through a database-generated ID or a human-readable ID, facilitating easy reference.

## Creating an API Template

To create an API template use `/api/assets` dashboard endpoint to create an Asset of kind `oas-template`.

## Applying Templates

### During API Import

1. **Specify Template ID**: Use the `templateID` parameter in the request. This can be the database ID or the human-readable ID of the template.
2. **Merge Process**: The request payload merges into the selected template. 
    * For maps like OAS paths and different components, keys are added alongside existing ones in the template, if there are any key already existing in the template, they are replaced (request payload has precedence).
    * For array properties such as `Servers` and `Tags`, values in the request replace those in the template.
    * If the `x-tyk-api-gateway` extension exists in the template, it is also applied to the newly created API.

Let's dive into an example where, during import we want to add a `/health` endpoint while importing an API. For that let's add a simple OAS document with `/health` endpoint as the template asset. This template also adds a Tyk ignore authentication middleware so that health check endpoint is not protected.

```
curl --location 'http://tyk-dashboard:3000/api/assets' \
--header 'Authorization: d244265dd51a4db55b77fad200198d85' \
--header 'Content-Type: application/json' \
--header 'Cookie: csrf_token=EWtTyjTrZoD0/cuFqsNPKcerjkqUGDTEMvdui4mHVFM=' \
--data '{
    "kind": "oas-template",
    "id": "healthcheck",
    "data": {
        "openapi": "3.0.0",
        "info": {
            "version": "1.0.0",
            "title": "Health",
            "license": {
                "name": "MIT"
            }
        },
        "paths": {
            "/health": {
                "get": {
                    "summary": "Health check endpoint.",
                    "operationId": "health",
                    "tags": [
                        "health"
                    ],
                    "responses": {
                        "200": {
                            "description": "Health check response",
                            "content": {
                                "application/json": {
                                    "schema": {
                                        "$ref": "#/components/schemas/Health"
                                    },
                                    "example": {
                                        "status": "Up"
                                    }
                                }
                            }
                        },
                        "503": {
                            "description": "Health check response",
                            "content": {
                                "application/json": {
                                    "schema": {
                                        "$ref": "#/components/schemas/Health"
                                    },
                                    "example": {
                                        "status": "Down"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        },
        "components": {
            "schemas": {
                "Health": {
                    "type": "object",
                    "properties": {
                        "status": {
                            "description": "Health check status",
                            "type": "string"
                        }
                    }
                }
            }
        },
        "x-tyk-api-gateway": {
            "info": {
                "name": "Health template",
                "state": {
                    "active": true
                }
            },
            "upstream": {
                "url": "http://petstore.swagger.io/v1"
            },
            "server": {
                "listenPath": {
                    "value": "/",
                    "strip": true
                }
            },
            "middleware": {
                "operations": {
                    "health": {
                        "ignoreAuthentication": {
                            "enabled": true
                        }
                    }
                }
            }
        }
    }
}'
```

Now, let's use this template while importing an OAS document. Let's use a petstore OAS example here. 

```
curl --location 'http://tyk-dashboard:3000/api/apis/oas/import?templateID=healthcheck' \
--header 'Authorization: d244265dd51a4db55b77fad200198d85' \
--header 'Content-Type: application/json' \
--header 'Cookie: csrf_token=EWtTyjTrZoD0/cuFqsNPKcerjkqUGDTEMvdui4mHVFM=' \
--data '{
  "openapi": "3.0.0",
  "info": {
    "version": "1.0.0",
    "title": "Swagger Petstore",
    "license": {
      "name": "MIT"
    }
  },
  "servers": [
    {
      "url": "http://petstore.swagger.io/v1"
    }
  ],
  "paths": {
    "/pets": {
      "get": {
        "summary": "List all pets",
        "operationId": "listPets",
        "tags": [
          "pets"
        ],
        "parameters": [
          {
            "name": "limit",
            "in": "query",
            "description": "How many items to return at one time (max 100)",
            "required": false,
            "schema": {
              "type": "integer",
              "maximum": 100,
              "format": "int32"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "A paged array of pets",
            "headers": {
              "x-next": {
                "description": "A link to the next page of responses",
                "schema": {
                  "type": "string"
                }
              }
            },
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Pets"
                }
              }
            }
          },
          "default": {
            "description": "unexpected error",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            }
          }
        }
      },
      "post": {
        "summary": "Create a pet",
        "operationId": "createPets",
        "tags": [
          "pets"
        ],
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/Pet"
              }
            }
          },
          "required": true
        },
        "responses": {
          "201": {
            "description": "Null response"
          },
          "default": {
            "description": "unexpected error",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            }
          }
        }
      }
    },
    "/pets/{petId}": {
      "get": {
        "summary": "Info for a specific pet",
        "operationId": "showPetById",
        "tags": [
          "pets"
        ],
        "parameters": [
          {
            "name": "petId",
            "in": "path",
            "required": true,
            "description": "The id of the pet to retrieve",
            "schema": {
              "type": "string"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Expected response to a valid request",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Pet"
                }
              }
            }
          },
          "default": {
            "description": "unexpected error",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "Pet": {
        "type": "object",
        "required": [
          "id",
          "name"
        ],
        "properties": {
          "id": {
            "type": "integer",
            "format": "int64"
          },
          "name": {
            "type": "string"
          },
          "tag": {
            "type": "string"
          }
        }
      },
      "Pets": {
        "type": "array",
        "maxItems": 100,
        "items": {
          "$ref": "#/components/schemas/Pet"
        }
      },
      "Error": {
        "type": "object",
        "required": [
          "code",
          "message"
        ],
        "properties": {
          "code": {
            "type": "integer",
            "format": "int32"
          },
          "message": {
            "type": "string"
          }
        }
      }
    }
  }
}'
```

This would result in the following API definition

```
{
  "components": {
    "schemas": {
      "Error": {
        "properties": {
          "code": {
            "format": "int32",
            "type": "integer"
          },
          "message": {
            "type": "string"
          }
        },
        "required": [
          "code",
          "message"
        ],
        "type": "object"
      },
      "Health": {
        "properties": {
          "status": {
            "description": "Health check status",
            "type": "string"
          }
        },
        "type": "object"
      },
      "Pet": {
        "properties": {
          "id": {
            "format": "int64",
            "type": "integer"
          },
          "name": {
            "type": "string"
          },
          "tag": {
            "type": "string"
          }
        },
        "required": [
          "id",
          "name"
        ],
        "type": "object"
      },
      "Pets": {
        "items": {
          "$ref": "#/components/schemas/Pet"
        },
        "maxItems": 100,
        "type": "array"
      }
    }
  },
  "info": {
    "license": {
      "name": "MIT"
    },
    "title": "Swagger Petstore",
    "version": "1.0.0"
  },
  "openapi": "3.0.0",
  "paths": {
    "/health": {
      "get": {
        "operationId": "health",
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "example": {
                  "status": "Up"
                },
                "schema": {
                  "$ref": "#/components/schemas/Health"
                }
              }
            },
            "description": "Health check response"
          },
          "503": {
            "content": {
              "application/json": {
                "example": {
                  "status": "Down"
                },
                "schema": {
                  "$ref": "#/components/schemas/Health"
                }
              }
            },
            "description": "Health check response"
          }
        },
        "summary": "Health check endpoint.",
        "tags": [
          "health"
        ]
      }
    },
    "/pets": {
      "get": {
        "operationId": "listPets",
        "parameters": [
          {
            "description": "How many items to return at one time (max 100)",
            "in": "query",
            "name": "limit",
            "schema": {
              "format": "int32",
              "maximum": 100,
              "type": "integer"
            }
          }
        ],
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Pets"
                }
              }
            },
            "description": "A paged array of pets",
            "headers": {
              "x-next": {
                "description": "A link to the next page of responses",
                "schema": {
                  "type": "string"
                }
              }
            }
          },
          "default": {
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            },
            "description": "unexpected error"
          }
        },
        "summary": "List all pets",
        "tags": [
          "pets"
        ]
      },
      "post": {
        "operationId": "createPets",
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/Pet"
              }
            }
          },
          "required": true
        },
        "responses": {
          "201": {
            "description": "Null response"
          },
          "default": {
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            },
            "description": "unexpected error"
          }
        },
        "summary": "Create a pet",
        "tags": [
          "pets"
        ]
      }
    },
    "/pets/{petId}": {
      "get": {
        "operationId": "showPetById",
        "parameters": [
          {
            "description": "The id of the pet to retrieve",
            "in": "path",
            "name": "petId",
            "required": true,
            "schema": {
              "type": "string"
            }
          }
        ],
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Pet"
                }
              }
            },
            "description": "Expected response to a valid request"
          },
          "default": {
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            },
            "description": "unexpected error"
          }
        },
        "summary": "Info for a specific pet",
        "tags": [
          "pets"
        ]
      }
    }
  },
  "servers": [
    {
      "url": "http://petstore.swagger.io/v1"
    }
  ],
  "x-tyk-api-gateway": {
    "info": {
      "id": "9971f2c773ae4157680182912f4a2f1f",
      "dbId": "65a90065a1ed7e4639d9cff6",
      "orgId": "65963968a1ed7e3f274d6a6b",
      "name": "Swagger Petstore",
      "state": {
        "active": true
      }
    },
    "upstream": {
      "url": "http://petstore.swagger.io/v1"
    },
    "server": {
      "listenPath": {
        "value": "/",
        "strip": true
      }
    },
    "middleware": {
      "operations": {
        "health": {
          "ignoreAuthentication": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

### During API Creation

- The process mirrors that of API import, with the addition of the `x-tyk-api-gateway` being a part of the API creation object and merging into the template definition.

## Limitations

- Templates are applicable only during the import or creation phase of an API. They cannot be applied post-creation.

---
