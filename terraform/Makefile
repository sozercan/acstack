SHELL += -eu

BLUE	:= \033[0;34m
GREEN	:= \033[0;32m
RED   := \033[0;31m
NC    := \033[0m

export DIR_KEY_PAIR   := .keypair
export DIR_SECRETS 		:= .secrets
export DIR_SSL        := .secrets
export DIR_KUBECONFIG := .kube

# CIDR_PODS: flannel overlay range
# - https://coreos.com/flannel/docs/latest/flannel-config.html
#
# CIDR_SERVICE_CLUSTER: apiserver parameter --service-cluster-ip-range
# - http://kubernetes.io/docs/admin/kube-apiserver/
#
# CIDR_VNET: VNET subnet
# - https://www.terraform.io/docs/providers/azurerm/r/virtual_network.html#address_prefix
#

# ∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨∨

export AZURE_LOCATION       		 ?= westus2
export KUBE_API_PUBLIC_FQDN      := $(CLUSTER_NAME).$(AZURE_LOCATION).cloudapp.azure.com

export AZURE_VM_KEY_NAME    ?= $(CLUSTER_NAME)
export AZURE_VM_KEY_PATH    := ${DIR_KEY_PAIR}/$(CLUSTER_NAME)/${AZURE_VM_KEY_NAME}.pem
# docker.io
# export AZURE_VHD_URI 				?= https://acstackimages.blob.core.windows.net/system/Microsoft.Compute/Images/acs-vhds/acstack-1526251964-osDisk.7fdd6d44-e3bd-4020-8033-47877b422c07.vhd
# cri/containerd/runc
export AZURE_VHD_URI 				?= https://acstackimages.blob.core.windows.net/system/Microsoft.Compute/Images/acs-vhds/acstack-1526252790-osDisk.5879f7c8-67f8-4c2e-a94d-95551fcc06db.vhd
export INTERNAL_TLD         := ${CLUSTER_NAME}.acs

export HYPERKUBE_IMAGE      ?= quay.io/coreos/hyperkube
export HYPERKUBE_TA

export CIDR_VNET            ?= 10.0.0.0/8
export CIDR_CONTROLLER      ?= 10.10.0.0/24
export CIDR_NODE        	  ?= 10.20.0.0/24
export CIDR_ETCD        	  ?= 10.30.0.0/24
export CIDR_DMZ        	    ?= 10.254.250.0/24
export CIDR_PODS            ?= 10.240.0.0/16
export CIDR_SERVICE_CLUSTER ?= 10.0.0.0/16

export K8S_SERVICE_IP       ?= 10.0.0.1
export K8S_DNS_IP           ?= 10.0.0.10
export KUBE_API_INTERNAL_IP	?= 10.10.0.250

export ETCD_IPS           	?= 10.30.0.10,10.30.0.11,10.30.0.12
export MASTER_IPS           ?= 10.240.0.247,10.240.0.248,10.240.0.249
export NODE_COUNT 					?= 1

# Alternative:
# CIDR_PODS ?= "172.15.0.0/16"
# CIDR_SERVICE_CLUSTER ?= "172.16.0.0/24"
# K8S_SERVICE_IP ?= 172.16.0.1
# K8S_DNS_IP ?= 172.16.0.10

# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ifndef CLUSTER_NAME
$(error CLUSTER_NAME is not set)
endif

.addons: ; @scripts/do-task "initialize add-ons" ./scripts/init-addons

## generate key-pair, variables and then `terraform apply`
all: prereqs create-keypair create-certs init apply
	@echo "${GREEN}✓ terraform portion of 'make all' has completed ${NC}\n"
	@$(MAKE) post-terraform

.PHONY: post-terraform
post-terraform:
	@$(MAKE) create-kubeconfig
	@$(MAKE) wait-for-cluster
	@$(MAKE) create-tls-bootstrap-config
	@$(MAKE) create-addons
	kubectl get nodes -o wide
	kubectl --namespace=kube-system get cs
	@echo "View nodes:"
	@echo "% make nodes"
	@echo "---"
	@echo "View uninitialized kube-system pods:"
	@echo "% make pods"
	@echo "---"
	@echo "Status summaries:"
	@echo "% make status"
	@echo "---"


## destroy and remove everything
clean: destroy delete-keypair
	@-pkill -f "kubectl proxy" ||:
	@-rm -rf build/${CLUSTER_NAME}
	@-rm -rf tmp ||:
	@-rm -rf ${DIR_SSL}/${CLUSTER_NAME} ||:
	@-kubectl config delete-cluster cluster-${CLUSTER_NAME}
	@-kubectl config delete-context ${CLUSTER_NAME}

## create tls bootstrap config
create-tls-bootstrap-config:
	@scripts/create-bootstrap-rbac
	@scripts/create-bootstrap-configmap
	@scripts/create-bootstrap-secret

## create kube-system addons
create-addons:
	scripts/create-default-storage-class
	scripts/create-kube-dns-service
	scripts/create-kube-system-configmap
	kubectl apply --recursive -f addons

create-admin-certificate: ; @scripts/do-task "create admin certificate" \
	scripts/create-admin-certificate

create-busybox: ; @scripts/do-task "create busybox test pod" \
	kubectl create -f test/pods/busybox.yml

create-kubeconfig: ; @scripts/do-task "create kubeconfig" \
	scripts/create-kubeconfig

## start proxy and open kubernetes dashboard
dashboard: ; @./scripts/dashboard

prereqs: ; @scripts/do-task "checking prerequisities" scripts/prereqs

## ssh into hostname=host
ssh: ; @scripts/ssh "ssh $(hostname).$(INTERNAL_TLD)"

## ssh into bastion host
ssh-bastion: ; @scripts/ssh

wait-for-cluster: ; @scripts/do-task "wait-for-cluster" scripts/wait-for-cluster

include makefiles/*.mk

.DEFAULT_GOAL := help
.PHONY: all clean create-addons create-admin-certificate create-busybox
.PHONY: delete-addons get-ca instances journal prereqs ssh ssh-bastion ssl
.PHONY: status test wait-for-cluster
