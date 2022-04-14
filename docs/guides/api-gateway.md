# Setting up Google Cloud Endpoints for your API Gateway

The following combines learning from the documentation for the [Cloud Endpoints Service](https://cloud.google.com/endpoints/docs) with context in the alis.exchange setup.

## Background
Google Cloud Endpoints allows us to setup API Gateways for our gRPC backend services enabling our service to be called via regular http requests. 
It makes use of a proxy layer (ESPv2) to enable scalability, monitoring and protection.

To setup endpoints, you first need to have backend gRPC services in place for which you want to setup the gateway.

## Initial Setup
Suppose we refer to the BookService gRPC service as outlined in the [Quick Start](https://docs.alis.exchange/getting-started/quick-start.html) of the docs.alis.exchange.
Our Backend service consists of 2 gRPC methods:
1. GetBook
2. ListBooks

These are contained within the `br` (Books Repository) product of the `foo` organisation on alis.exchange.

To setup up our gateway, we will create a new neuron titled: `services-gateway-v1`. This neuron will allow us to bundle the terraform, Dockerfile and config files conveniently under the same neuron.

In the case of the `br` product for the `foo` organisation, we would create the API gateways neuron with the following:
`alis neuron create foo.br.services-gateway-v1`

When prompted whether you would like to add an environment variable, respond with `y`. Then, assign the name `ALIS_OS_HASH` with value `tbc`. After deploying our service, we will be able to change this value at a later stage.

Before doing anything else, we need to enable the endpoints service for our project. We do this by adding the following lines to the `apis.tf` file in the `proto/foo/br/services/gateway/v1` directory     to enable the endpoints service:
```terraform
resource "google_project_service" "endpoints_googleapis_com" {
  project = google_project.product_deployment.project_id
  service = "endpoints.googleapis.com"
}
```

We then need to rebuild and redeploy our product to accommodate the incorporation of this service. Commit the changes in `apis.tf` and run `alis product build foo.br` followed by `alis product deploy foo.br`.

## Configuration
To configure the endpoints, we require 2 files:
1. The protobuf descriptor file
2. An API config file

The protobuf descriptor file is a file that describes the gRPC services which are available for an entire product. This is obtained 
by running the `alis gen descriptor foo.br` command for the `foo.br` product in particular.

After executing this command, one will find a `descriptor.pb` file sitting at the product level in the `proto` folder.  

The API config file is specified by us to determine exactly the gRPC services which we want to make available, where the backends exist for these services and specifying usage for the services. In the proto folder, add the following `endpoints.tf` file in the directory `services/gateway/v1/`:
```terraform
resource "google_api_gateway_api" "http_transcoder" {
  project = var.ALIS_OS_PRODUCT_PROJECT
  provider = google-beta
  api_id = var.ALIS_OS_PROJECT
  display_name = "HTTP Transcoder | ${var.ALIS_OS_PROJECT}"
}

resource "local_file" "api_config" {
  content = yamlencode({
    type: "google.api.Service"
    config_version: 3
    name: "services-gateway-v1-${var.ALIS_OS_HASH}-ew.a.run.app" # host name
    title: "BR | ${var.ALIS_OS_PROJECT}"
    apis:[
      {
        name: "foo.br.resources.books.v1.BooksService"
      },
    ]
    backend: {
      rules: [
        {
          address: "grpcs://resources-books-v1-${var.ALIS_OS_HASH}-ew.a.run.app"
          disable_auth: true
          deadline: 120
          selector: "foo.br.resources.books.v1.*"
        },
      ]
    }
    usage: {
      rules: [
        {
          selector: "*"
          allow_unregistered_calls: true
        }
      ]
    }
  })
  filename = "${path.module}/api_config.yaml"
}

# Used for reserving the cloud run host name
resource "google_cloud_run_service" "reserve_cloud_run_host" {
  provider = google-beta
  name     = var.ALIS_OS_NEURON
  location = "europe-west1"

  template {
    spec {
      containers {
        image = "gcr.io/cloudrun/hello"
      }
      service_account_name = "alis-exchange@${var.ALIS_OS_PROJECT}.iam.gserviceaccount.com"
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }
}

# Deploys the api config files to our gateway service
resource "google_endpoints_service" "grpc_service" {
  service_name         = "services-gateway-v1-${var.ALIS_OS_HASH}-ew.a.run.app"
  project              = var.ALIS_OS_PROJECT
  grpc_config          = local_file.api_config.content
  protoc_output_base64 = filebase64("descriptor.pb")
  depends_on = [module.gcloud_config]
}

output "config_id" {
  value = google_endpoints_service.grpc_service.config_id
}

output "dns_address" {
  value = google_endpoints_service.grpc_service.dns_address
}

output "apis" {
  value = google_endpoints_service.grpc_service.apis
}

output "endpoints" {
  value = google_endpoints_service.grpc_service.endpoints
}
```

We specify our API config yaml file within the terraform to allow us to inject the required environment variables into the config. Ensure you add the following line to the `variables.tf` file:
```terraform
# Custom neuron specific ENVS:
variable "ALIS_OS_HASH" {}
```
Delete all .go files in the `product` directory in `services/gateway/v1` and replace the content of the Dockerfile with:
```dockerfile
FROM gcr.io/cloudrun/hello
```

Commit all files in the `proto` folder in `services/gateway/v1` file. Then build and deploy the neuron, using the `-e` flag to set the `ALIS_OS_HASH` variable to the value of the cloud run hash (this can be found by looking at the cloud run of other services you have deployed).
The logs of the cloud build will contain the values specified by the output of the above terraform. Take note of the value of `config_id`, which we will need for setting up the Dockerfile.
## Setting up the Dockerfile
Our backend services are now set up and configured. The last step is to set up the Dockerfile with the ESPv2 image and correct configuration. Please replace the value of ALIS_OS_HASH and CONFIG_ID accordingly.

```dockerfile
FROM gcr.io/google.com/cloudsdktool/google-cloud-cli:latest as gcloudbuild

WORKDIR ./

RUN curl --fail -o "service.json" -H "Authorization: Bearer $(gcloud auth print-access-token)" \
          "https://servicemanagement.googleapis.com/v1/services/services-gateway-v1-{ALIS_OS_HASH}-ew.a.run.app/configs/{CONFIG_ID}?view=FULL"

FROM gcr.io/endpoints-release/endpoints-runtime-serverless:2

USER root
ENV ENDPOINTS_SERVICE_PATH /etc/endpoints/service.json
COPY --from=gcloudbuild ./service.json ${ENDPOINTS_SERVICE_PATH}

RUN ls -a
RUN cat ${ENDPOINTS_SERVICE_PATH}

RUN chown -R envoy:envoy ${ENDPOINTS_SERVICE_PATH} && chmod -R 755 ${ENDPOINTS_SERVICE_PATH}
USER envoy

RUN ls -a
ENTRYPOINT ["/env_start_proxy.py"]
```

Rebuild and redeploy the neuron once more and your service will be set up!
