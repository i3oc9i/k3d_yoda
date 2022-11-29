SHELL := /bin/bash

KLUSTER_NAME = yoda

KLUSTER_URL  = k3d-yoda.local
REGISTRY_URL = registry-yoda.local:5000

all: help

k3d-up:
	k3d cluster create $(KLUSTER_NAME) --config ./confg/default.yaml
	kubectl cluster-info
	kubectl get node -o wide

k3d-start:
	k3d cluster start $(KLUSTER_NAME) 

k3d-stop:
	k3d cluster stop $(KLUSTER_NAME) 

k3d-down:
	k3d cluster delete $(KLUSTER_NAME) 

# ------------------------------------------------------------------- Help
help:
	@echo "make k3d-up   - create the playground k3d kluster and the registry."
	@echo "make k3d-down - destroy the playground k3d kluster and the registry."
