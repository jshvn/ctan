# The pipeline's toolbox: the same versions GitHub's ubuntu-latest runner ships (rsync
# 3.2.7, gnupg 2.4.4, xz 5.4.5, curl 8.5.0), so a run here and a run in Actions differ
# only in wall clock. Base pinned by digest; the tag is informational.
# pinned: 2026-08-26 -- refresh by re-running `container image pull ubuntu:24.04` and
# copying the first digest `container image inspect ubuntu:24.04` prints.
FROM ubuntu:24.04@sha256:33ceb71981b602c1a7443a53469e4dba065f7503eab3078a2d7a57a2ab987517

ARG TASK_VERSION=3.53.1
ARG AWSCLI_VERSION=2.36.24

# perl carries shasum; unzip is only for the AWS CLI installer.
# Each RUN deletes its own scratch: a layer keeps whatever it leaves. Lists are 53 MB.
RUN apt-get update && apt-get install -y --no-install-recommends \
      rsync gnupg xz-utils curl ca-certificates perl unzip \
 && rm -rf /var/lib/apt/lists/*

# task: the release tarball, checksum from task_checksums.txt of the same release.
# Bumping TASK_VERSION means bumping both sums.
RUN set -eu; arch=$(uname -m); case "$arch" in \
      aarch64) t=arm64; sum=e3ad19101493a0112e1f22ae8ccc54bf03e533b1076a0ca1e6c782a09ad2e588 ;; \
      x86_64)  t=amd64; sum=a54a408f6861ff921f6e87774180db31bacd8c1e7c944ca696db9fea49a82fc7 ;; \
      *) echo "unsupported arch $arch" >&2; exit 1 ;; esac; \
    curl -fsSL -o /tmp/task.tgz "https://github.com/go-task/task/releases/download/v${TASK_VERSION}/task_linux_${t}.tar.gz"; \
    echo "$sum  /tmp/task.tgz" | sha256sum -c -; \
    tar -xzf /tmp/task.tgz -C /usr/local/bin task; \
    rm -f /tmp/task.tgz; \
    task --version

# AWS CLI v2 from the official installer, version pinned; ponytail: the zip's PGP
# signature is not checked (it would need AWS's key in the image); HTTPS and the pin are
# the guarantee.
# The zip and its unpacked installer are 320 MB of scratch once `./aws/install` returns.
# ponytail: botocore ships 432 service models and the pipeline calls one, so all but s3,
# sts and the data-root *.json (endpoints, partitions) go -- 105 MB. Ceiling: an `aws`
# call to any other service dies on a model lookup; add it to the keep list, or drop the
# trim. `examples/` and `topics/` need groff to render, `ac.index` is completion only.
# The `s3 ls` at a dead port is the check -- reaching the network means s3 still loads.
RUN set -eu; arch=$(uname -m); \
    curl -fsSL -o /tmp/awscli.zip "https://awscli.amazonaws.com/awscli-exe-linux-${arch}-${AWSCLI_VERSION}.zip"; \
    cd /tmp && unzip -q awscli.zip && ./aws/install; \
    rm -rf /tmp/awscli.zip /tmp/aws; \
    d=/usr/local/aws-cli/v2/current/dist/awscli; \
    mkdir /tmp/keep; \
    for k in s3 sts; do mv "$d/botocore/data/$k" /tmp/keep/; done; \
    mv "$d/botocore/data"/*.json /tmp/keep/; \
    rm -rf "$d/botocore/data"; mv /tmp/keep "$d/botocore/data"; \
    rm -rf "$d/examples" "$d/topics" "$d/data/ac.index"; \
    aws --version; \
    aws s3 ls s3://x --no-sign-request --endpoint-url http://127.0.0.1:1 2>&1 \
      | grep -qiE 'connect|endpoint|refused'

WORKDIR /work
