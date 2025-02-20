SHELL := /bin/bash

AWS_ACCOUNT_ID?=$(shell aws sts get-caller-identity --query Account --output text)
AWS_REGION?=us-west-2

BASE_IMAGE?=public.ecr.aws/amazonlinux/amazonlinux:2

IMAGE_REPO?=$(AWS_ACCOUNT_ID).dkr.ecr.$(AWS_REGION).amazonaws.com
IMAGE_NAME?=eks-distro-base
IMAGE_TAG?=$(shell date "+%F-%s")
IMAGE?=$(IMAGE_REPO)/$(IMAGE_NAME):$(IMAGE_TAG)

DOCKERFILE?=Dockerfile
PACKAGES?=

MAKE_ROOT=$(shell cd "$(shell dirname "${BASH_SOURCE[0]}")" && pwd -P)

ifeq ("$(REPO_OWNER)","")
    $(error No org information was provided, please set and export REPO_OWNER environment variable. \
      This is used to raise a pull request against your org after updating tags in the respective files.)
endif

.PHONY: buildkit-check
buildkit-check:
	$(MAKE_ROOT)/../scripts/buildkit_check.sh

.PHONY: local-images
local-images: buildkit-check
	./check_update.sh $(IMAGE_TAG) $(PACKAGES)
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt filename=$(DOCKERFILE) \
		--opt platform=linux/amd64 \
		--opt build-arg:BASE_IMAGE=$(BASE_IMAGE) \
		--local dockerfile=./ \
		--local context=. \
		--output type=oci,oci-mediatypes=true,name=$(IMAGE),dest=/tmp/$(IMAGE_NAME).tar

.PHONY: images
images: buildkit-check
	if [ "$(JOB_TYPE)" = "postsubmit" ]; then \
		./check_update.sh $(IMAGE_TAG) $(PACKAGES) ; \
	fi
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt filename=$(DOCKERFILE) \
		--opt platform=linux/amd64,linux/arm64 \
		--opt build-arg:BASE_IMAGE=$(BASE_IMAGE) \
		--local dockerfile=./ \
		--local context=. \
		--output type=image,oci-mediatypes=true,name=$(IMAGE),push=true

# Update tag files in and create PR against eks-distro-build-tooling and eks-distro repos
.PHONY: create-pr
create-pr:
	if [ "$(JOB_TYPE)" = "presubmit" ] || [ "$(JOB_TYPE)" = "postsubmit" ]; then \
		./update_base_image.sh $(IMAGE_TAG); \
	elif [ "$(shell cat $(MAKE_ROOT)/eks-distro-base-pushed)" = "true" ] || [ "$(shell cat $(MAKE_ROOT)/eks-distro-minimal-base-pushed)" = "true" ]; then \
		./update_base_image.sh $(IMAGE_TAG); \
	fi

.PHONY: docker-public-login
docker-public-login:
	aws --region us-east-1 ecr-public get-authorization-token --output=text \
	 --query 'authorizationData.authorizationToken' |  \
	 base64 --decode | cut -d: -f2 | \
	 docker login -u AWS --password-stdin https://public.ecr.aws

.PHONY: public-images
public-images: docker-public-login
	$(eval REPOSITORY=$(shell aws --region us-east-1 ecr-public describe-registries --query 'registries[*].registryUri' --output text))
	$(eval IMAGE=$(REPOSITORY)/$(IMAGE_NAME):$(IMAGE_TAG))
	buildctl \
		build \
		--frontend dockerfile.v0 \
		--opt filename=$(DOCKERFILE) \
		--opt platform=linux/amd64,linux/arm64 \
		--opt build-arg:BASE_IMAGE=$(BASE_IMAGE) \
		--local dockerfile=./ \
		--local context=. \
		--output type=image,oci-mediatypes=true,name=$(IMAGE),push=true

# for local development only
.PHONY: ecr
ecr:
	aws ecr create-repository \
		--repository-name $(IMAGE_NAME) \
		--image-tag-mutability IMMUTABLE \
		--image-scanning-configuration  scanOnPush=true || true

.PHONY: docker
docker:
	docker build \
		-t $(IMAGE) \
		--build-arg BASE_IMAGE=$(BASE_IMAGE) \
		--build-arg TARGETARCH=amd64 \
		--build-arg TARGETOS=linux \
		-f $(DOCKERFILE) .

.PHONY: docker-push
docker-push: docker
	docker push $(IMAGE)

.PHONY: build
build: local-images

.PHONY: release
release: images

.PHONY: update
update: buildkit-check
	echo "false" > $(MAKE_ROOT)/$(IMAGE_NAME)-pushed
	$(eval RETURN_MESSAGE="$(shell ./check_update.sh $(IMAGE_TAG) $(PACKAGES) )")
	if [ $(RETURN_MESSAGE) = "Updates required" ]; then \
		source $(MAKE_ROOT)/../scripts/setup_public_ecr_push.sh && $(MAKE) images; \
		echo "true" > $(MAKE_ROOT)/$(IMAGE_NAME)-pushed; \
	elif [ $(RETURN_MESSAGE) = "Error" ]; then \
		exit 1; \
	fi

.PHONY: all
all: release
