## OpenShift Grafana Cartridge

A simple cartridge to give access to the app's metrics through Grafana. We inject some
environment variables in the Grafana source to change where it looks for metrics, my
goal is to push a change upstream to make it cleaner.

Currently the cartridge uses OpenShift environment variables to restrict access to
a subset of the metrics, this doesn't prevent a user to access to other applications
metrics and isn't secure at all. Access control can only be managed on Graphite's end,
Grafana is a pure client-side application.

Before installing you will need to edit `html/config.js` to point to your Graphite and
Elasticsearch servers, and `env/OPENSHIFT_GRAFANA_METRICS_PREFIX` to set the prefix
you want to use (eg. `openshift.metrics`).
