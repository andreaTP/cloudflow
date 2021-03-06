# Build the Cloudflow documentation

include definitions.mk

SHELL_DIR := $(shell dirname $(realpath $(lastword $(MAKEFILE_LIST))))
ROOT_DIR := ${SHELL_DIR}/..

antora_docker_image     := antora/antora
antora_docker_image_tag := 2.3.1

work_dir := ${ROOT_DIR}/docs/target

staging_dir := ${work_dir}/staging

javascaladoc_dir := ${staging_dir}/docs/current/api

all: build

local-preview: html-author-mode
	@echo "Access the documentation on http://localhost:8000"
	(cd target/staging/; python3 -m http.server)


show:
	echo work dir: ${work_dir}
	echo ROOT_DIR: ${ROOT_DIR}

clean:
	rm -rf ${work_dir}

build: clean html javascaladoc_staged print-site

get-version:
	# get latest tag
	$(eval version=$(shell sh -c "git for-each-ref refs/tags --sort=-taggerdate --format='%(refname)' --count=1 | sed 's|refs/tags/v||'"))
	# query bintray for the corresponding artifact version
	$(eval bintray_version=$(shell sh -c "wget -qO - https://api.bintray.com/packages/lightbend/cloudflow-cli/kubectl-cloudflow | jq -r '.versions[] | select( . | startswith(\"${version}.\") )'"))

set-antora-versions: get-version
	yq write shared-content-source/docs/base-antora.yml version "${version}" > shared-content-source/docs/antora.yml
	yq write ../examples/snippets/base-antora.yml version "${version}" > ../examples/snippets/antora.yml

	yq write docs-source/base-author-mode-site.yml 'asciidoc.attributes.cloudflow-version' "${version}" > docs-source/author-mode-site.yml
	yq write -i docs-source/author-mode-site.yml 'asciidoc.attributes.cloudflow-examples-version' "${version}"
	yq write -i docs-source/author-mode-site.yml 'asciidoc.attributes.cloudflow-branch-version' "v${version}"
	yq write -i docs-source/author-mode-site.yml 'asciidoc.attributes.kubectl-plugin-version' "${bintray_version}"

	yq write docs-source/base-site.yml 'asciidoc.attributes.cloudflow-version' "${version}" > docs-source/site.yml
	yq write -i docs-source/site.yml 'asciidoc.attributes.cloudflow-examples-version' "${version}"
	yq write -i docs-source/site.yml 'asciidoc.attributes.cloudflow-branch-version' "v${version}"
	yq write -i docs-source/site.yml 'asciidoc.attributes.kubectl-plugin-version' "${bintray_version}"

html: clean
	docker run \
		-u $(shell id -u):$(shell id -g) \
		--privileged \
		-v ${ROOT_DIR}:/antora \
		--rm \
		-t ${antora_docker_image}:${antora_docker_image_tag} \
		--cache-dir=./.cache/antora \
		--stacktrace \
		docs/docs-source/site.yml
	@echo "Done"

html-author-mode: clean
	docker run \
		-u $(shell id -u):$(shell id -g) \
		-v ${ROOT_DIR}:/antora \
		--rm \
		-t ${antora_docker_image}:${antora_docker_image_tag} \
		--cache-dir=./.cache/antora \
		--stacktrace \
		docs/docs-source/author-mode-site.yml
	@echo "Done"

check-links:
	docker run \
		-v ${ROOT_DIR}:/antora \
		--rm \
		--entrypoint /bin/sh \
		-t lightbend/antora-doc:0.1.0 \
		--cache-dir=./.cache/antora \
		-c 'find docs/docs-source -name '*.adoc' -print0 | xargs -0 -n1 asciidoc-link-check -p -c docs/docs-source/asciidoc-link-check-config.json'

list-todos: html
	docker run \
		-v ${ROOT_DIR}:/antora \
		--rm \
		--entrypoint /bin/sh \
		-t ${antora_docker_image}:${antora_docker_image_tag} \
		--cache-dir=./.cache/antora \
		-c 'find /antora/docs-source/build/site/cloudflow/${version} -name "*.html" -print0 | xargs -0 grep -iE "TODO|FIXME|REVIEWERS|adoc"'

# Generate the ScalaDoc and the JavaDoc, and put it in ${output}/scaladoc and ${output}/javadoc
javascaladoc: 
	cd ${ROOT_DIR}/core && sbt clean unidoc

javascaladoc_staged: ${javascaladoc_dir} javascaladoc
	cp -r ${ROOT_DIR}/core/target/scala-2.12/unidoc ${javascaladoc_dir}/scaladoc
	cp -r ${ROOT_DIR}/core/target/javaunidoc ${javascaladoc_dir}/javadoc

${work_dir}: 
	mkdir -p ${work_dir}

${staging_dir}:
	mkdir -p ${staging_dir}

${javascaladoc_dir}: 	
	mkdir -p ${javascaladoc_dir}/scaladoc
	mkdir -p ${javascaladoc_dir}/javadoc

print-site:
	# The result directory with the contents of this build:
	@echo "${staging_dir}"
