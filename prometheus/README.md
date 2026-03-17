Put your Prometheus configuration here.
If you change anything here or in the scrape_configs, you need to reload Prometheus.

You can either use the API:

    curl -X POST http://localhost:9090/-/reload

Or just signal the container:

    docker kill --signal=SIGHUP prometheus
