Put your scrape_configs here.

Two sample files are included: one named `infra.yml` that scrapes our own instances of Vector, Loki, Grafana and Prometheus, and one named `rtb_tour.yml` meant to be used with the RtBrick Tour VM (or generally with RtBrick nodes).

If you want to use this last config, change all IP addresses of targets to the addres of the RtBrick tour VM, as this VM uses a single ctrld instance for each node.

if you scrape multiple nodes, instead, change each target to the corresponding address.
