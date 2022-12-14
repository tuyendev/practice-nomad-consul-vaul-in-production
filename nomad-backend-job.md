# DEPLOY BACKEND JOB

1. Deploy job 


```hcl
job "appservice" {
  datacenters = ["saigon"]
  type        = "service"
  group "demo-backend" {
    count = 3
    scaling {
      min     = 2
      max     = 5
      enabled = true
      policy {
        evaluation_interval = "2s"
        cooldown            = "5s"
        check "cpu_usage" {
          source = "prometheus"
          query  = "avg(nomad_client_allocs_cpu_total_percent{task='server'})"
          strategy "target-value" {
            target = 10
          }
        }
      }
    }
    task "server" {
      driver = "docker"
      # Node monitor for monitoring job only
      constraint {
        attribute = "${node.class}"
        operator  = "!="
        value     = "monitor"
      }
      config {
        image              = "tuyendev/nomad-app-demo:v1"
        image_pull_timeout = "10m"
        port_map {
          http = 8080
        }
        logging {
          type = "fluentd"
          config {
            fluentd-address = "${attr.unique.network.ip-address}:24224"
            tag = "backend-${NOMAD_ALLOC_NAME}"
          }
        }
      }
      resources {
        memory = 1024
        cpu    = 100
        network {
          mbits = 10
          port "http" {}
        }
      }
      service {
        name = "backend-service"
        port = "http"
        tags = [
          "demo",
          "traefik.enable=true",
          "traefik.http.routers.backend-route.entrypoints=backend",
          "traefik.http.routers.backend-route.service=backend-service", 
          "traefik.http.routers.backend-route.rule=PathPrefix(`/`)",
          "traefik.http.routers.backend-route.middlewares=backend-block-management",
          "traefik.http.middlewares.backend-block-management.replacepathregex.regex=^/management(.*)",
          "traefik.http.middlewares.backend-block-management.replacepathregex.replacement=/blocked/$1",
        ]
        check {
          type     = "http"
          path     = "/management/health"
          interval = "2s"
          timeout  = "2s"
        }
      }
    }
  }
}
```