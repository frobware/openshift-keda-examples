IMAGE_TAG ?= amcdermo/nodes-ready-app

export GOPROXY=https://proxy.golang.org,direct

build:
	CGO_ENABLED=0 go build -mod=vendor -o nodes-ready-app .

image: fmt vet build
	$(RM) nodes-ready-app
	podman build -f Containerfile -t $(IMAGE_TAG)

push-image:
	podman tag $(IMAGE_TAG) quay.io/$(IMAGE_TAG)
	podman push quay.io/$(IMAGE_TAG)

fmt:
	go fmt ./...

vet:
	go vet ./...

$(HOME)/.config/containers/policy.json: policy.json.example | $(HOME)/.config/containers
	install -m644 $< $@

$(HOME)/.config/containers:
	mkdir -p $@

.PHONY: build push-image fmt vet
