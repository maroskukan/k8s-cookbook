#!/usr/bin/env make

run_website:
	docker build -t explorecalifornia.com . && \
		docker run -p 8080:80 -d --name explorecalifornia.com --rm explorecalifornia.com

stop_website:
	docker stop explorecalifornia.com

install_kind:
	curl -sLO https://github.com/kubernetes-sigs/kind/releases/download/v0.24.0/kind-linux-amd64 && \
		install kind-linux-amd64 ./kind && rm kind-linux-amd64 && \
		kind --version

create_kind_cluster: install_kind install_kubectl create_docker_registry
	kind create cluster --name explorecalifornia.com --config ./kind_config.yaml --image=kindest/node:v1.21.12 || true && \
		kubectl get nodes

install_kubectl:
	curl -sLO https://dl.k8s.io/release/v1.21.12/bin/linux/amd64/kubectl && \
		chmod +x kubectl

create_docker_registry:
	if docker ps | grep -q 'local-registry'; \
	then echo "---> local-registry already created; skipping"; \
	else docker container run --name local-registry -d --restart=always -p 5000:5000 registry:2; \
	fi

connect_registry_to_kind_network:
	docker network connect kind local-registry || true

connect_registry_to_kind: connect_registry_to_kind_network
	kubectl apply -f ./kind_configmap.yaml

create_kind_cluster_with_registry:
	$(MAKE) create_kind_cluster && $(MAKE) connect_registry_to_kind

delete_docker_registry:
	docker container stop local-registry && docker container rm local-registry

delete_kind_cluster: delete_docker_registry
	kind delete cluster --name explorecalifornia.com
