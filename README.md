## Grafana

This repository contains Grafana configuration and deployment resources.

## Overview

Grafana is an open source platform for monitoring and observability. It allows you to query, visualize, alert on, and explore your metrics, logs, and traces.

Integration with the Openshift outh-proxy means that our authentication is managed by the same methods as used in the Openshift that Grafana is running in.

Client interaction works by HTTPS - terminated at ingress - to the Oauth Proxy. Following successfull signon, this redirect traffic to port 3000 in the same pod - Grafana.

## Usage

1. Install Grafana or deploy it in your environment.
2. Add your data sources.
3. Import or create dashboards.

## Notes

1. Install
* oc login -u xxx -p yyy https://apiserver:5443
* oc new-project grafana
* oc create -f grafana-pvc.yaml
* oc create -f grafana-serviceaccount.yaml
  - note the annotation - this allows the grafana-oauth route to redirect to grafana with a successful authentication
  - see the section on oauth-proxy, below
* oc adm policy add-scc-to-user anyuid -z grafana -n grafana
* oc adm policy add-cluster-role-to-user cluster-monitoring-view -z grafana -n grafana
* oc create -f grafana-oauth-proxy-secret.yaml
* oc create -f grafana-statefulset.yaml
* oc create -f grafana-service.yaml
* oc create -f grafana-route.yaml


--------------------

This was part of the original, before we merged the oauth-proxy onto the service and route. It's useful to keep these notes, especially to remind me how to create the session_secret!

2. Initial logon
* User = admin, Password = admin
* Immediately prompted to change admin password - which we did to 'sausage'

3. Integrate oauth-proxy for Prometheus user based logins
* Create the cookie-secret:
  - python -c 'import os,base64; print(base64.b64encode(os.urandom(128)))'
  - Put the output in the value for session_secret in the grafana-oauth-proxy-secret.yaml
  - oc create -f grafana-oauth-proxy-secret.yaml
* Create the new service, referencing port 3001 (oauth-proxy)
  - oc create -f grafana-oauth-service.yaml
* Create the new route, referencing the oauth-service
  - oc create -f grafana-oauth-route.yaml
* Recreate the stateful set

Things I learnt:
1. Port names for pods can't be more than 15 characters
2. If you want to use an imagestream from another namespace, you need to use an annotation. Here's the one we used for oauth-proxy:
   image.openshift.io/triggers: '[{"from":{"kind":"ImageStreamTag","name":"oauth-proxy:v4.4","namespace":"openshift"},"fieldPath":"spec.template.spec.containers[?(@.name==\"grafana-proxy\")].image"}]'
   Not that this actually works, because you have to give an "image" stanza, and it can't just contain a white space.
3. Read lots of other ways of getting to the same error, and in the end, tracked down oauth-proxy on https://catalog.redhat.com and used the container path provided there. Works.
4. The serviceaccount that oauth-proxy runs with needs to have an annotation applied to allow it to redirect traffic to a different route. See grafana-serviceaccount.yaml
