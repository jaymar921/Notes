# Running and Configuring Prometheus

### Download Prometheus

Check [Prometheus download options](https://prometheus.io/download/) including OS and CPI architecture).

> Run on Windows:

```
$ProgressPreference = "SilentlyContinue"; `
iwr -useb `
  -o prometheus-2.18.1.windows-amd64.tar.gz `

https://github.com/prometheus/prometheus/releases/download/v2.18.1/prometheus-2.18.1.windows-amd64.tar.gz

tar xvfz prometheus.tar.gz
```

Explore `prometheus.yml`

- alerting and rules
- delete all except global and scrape config

### Run Prometheus with default config

Switch to the download folder and run the binary:

```
cd prometheus-2.18.1.windows-amd64

./prometheus.exe
```

Check the logs output

- Starting TSDB
- Loading configuration file filename=prometheus.yml
- Server is ready to receive web requests

And the `data` directory

### Explore the Prometheus UI

Check the Prometheus' own metrics at http://localhost:9090/metrics

Browse the UI at http://localhost:9090

Check the Status pages:

- Configuration
- Command-Line Flags
- Targets
- Service Discovery

And some data in the Graph page:

- `go_info`, standard Go info with instance and job labels
- `promhttp_metric_handler_requests_total`, graph (UI shows number of time series)

### Prometheus Config file

```yml
global:
  scrape_interval: 30s

scrape_configs:
  - job_name: "linux"
    static_configs:
      - targets: ["localhost:9100"]

  - job_name: "windows"
    static_configs:
      - targets: ["localhost:8080"]
```
