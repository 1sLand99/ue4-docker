{% if combine %}
FROM prerequisites as source
{% else %}
ARG NAMESPACE
ARG PREREQS_TAG
FROM ${NAMESPACE}/ue4-build-prerequisites:${PREREQS_TAG}
{% endif %}

{% if source_mode == "copy" %}

# Copy the Unreal Engine source code from the host system
ARG SOURCE_LOCATION
COPY --chown=ue4:ue4 ${SOURCE_LOCATION} /home/ue4/UnrealEngine

{% else %}

# The git repository that we will clone
ARG GIT_REPO=""

# The git branch/tag/commit that we will checkout
ARG GIT_BRANCH=""

{% if credential_mode == "secrets" %}

# Install our git credential helper that retrieves credentials from build secrets
COPY --chown=ue4:ue4 git-credential-helper-secrets.sh /tmp/git-credential-helper-secrets.sh
ENV GIT_ASKPASS=/tmp/git-credential-helper-secrets.sh
RUN chmod +x /tmp/git-credential-helper-secrets.sh

# Clone the UE4 git repository using the build secret credentials
# (Note that we include the changelist override value here to ensure any cached source code is invalidated if
#  the override is modified between runs, which is useful when testing preview versions of the Unreal Engine)
ARG CHANGELIST
RUN --mount=type=secret,id=username,uid=1000,required \
	--mount=type=secret,id=password,uid=1000,required \
	CHANGELIST="$CHANGELIST" \
	mkdir /home/ue4/UnrealEngine && \
	cd /home/ue4/UnrealEngine && \
	git init && \
	{% if git_config %}
	{% for key, value in git_config.items() %}
	git config {{ key }} {{ value }} && \
	{% endfor %}
	{% endif %}
	git remote add origin "$GIT_REPO" && \
	git fetch --progress --depth 1 origin "$GIT_BRANCH" && \
	git checkout FETCH_HEAD

{% else %}

# Retrieve the address for the host that will supply git credentials
ARG HOST_ADDRESS_ARG=""
ENV HOST_ADDRESS=${HOST_ADDRESS_ARG}

# Retrieve the security token for communicating with the credential supplier
ARG HOST_TOKEN_ARG=""
ENV HOST_TOKEN=${HOST_TOKEN_ARG}

# Install our git credential helper that forwards requests to the credential HTTP endpoint on the host
COPY --chown=ue4:ue4 git-credential-helper-endpoint.sh /tmp/git-credential-helper-endpoint.sh
ENV GIT_ASKPASS=/tmp/git-credential-helper-endpoint.sh
RUN chmod +x /tmp/git-credential-helper-endpoint.sh

# Clone the UE4 git repository using the endpoint-supplied credentials
RUN mkdir /home/ue4/UnrealEngine && \
	cd /home/ue4/UnrealEngine && \
	git init && \
	{% if git_config %}
	{% for key, value in git_config.items() %}
	git config {{ key }} {{ value }} && \
	{% endfor %}
	{% endif %}
	git remote add origin "$GIT_REPO" && \
	git fetch --progress --depth 1 origin "$GIT_BRANCH" && \
	git checkout FETCH_HEAD

{% endif %}

{% endif %}

{% if not disable_all_patches %}
# Enable verbose output for steps that patch files?
ARG VERBOSE_OUTPUT=0
{% endif %}

{% if (not disable_all_patches) and (not disable_release_patches) %}
# Apply our bugfix patches to broken Engine releases
# (Make sure we do this before the post-clone setup steps are run)
COPY --chown=ue4:ue4 patch-broken-releases.py /tmp/patch-broken-releases.py
RUN python3 /tmp/patch-broken-releases.py /home/ue4/UnrealEngine $VERBOSE_OUTPUT
{% endif %}

# Run post-clone setup steps, ensuring our package lists are up to date since Setup.sh doesn't call `apt-get update`
{% if credential_mode == "secrets" %}

# Ensure Setup.sh uses the same cache path when building either UE4 or UE5
ENV UE_GITDEPS=/home/ue4/gitdeps
ENV UE4_GITDEPS=/home/ue4/gitdeps
RUN mkdir "$UE_GITDEPS"

# When running with BuildKit, we use a cache mount to cache the dependency data across multiple build invocations
WORKDIR /home/ue4/UnrealEngine
RUN --mount=type=cache,target=/home/ue4/gitdeps,uid=1000,gid=1000 sudo apt-get update && \
	./Setup.sh {{ gitdependencies_args }} && \
	sudo rm -rf /var/lib/apt/lists/*

{% else %}

# When running without BuildKit, we use the `-no-cache` flag to disable caching of dependency data in `.git/ue4-gitdeps`, saving disk space
WORKDIR /home/ue4/UnrealEngine
RUN sudo apt-get update && \
	./Setup.sh -no-cache {{ gitdependencies_args }} && \
	sudo rm -rf /var/lib/apt/lists/*

{% endif %}

# Set the changelist number in Build.version to ensure our Build ID is generated correctly
ARG CHANGELIST
COPY set-changelist.py /tmp/set-changelist.py
RUN python3 /tmp/set-changelist.py /home/ue4/UnrealEngine/Engine/Build/Build.version $CHANGELIST
