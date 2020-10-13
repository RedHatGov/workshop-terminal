FROM registry.access.redhat.com/ubi8/python-36:latest

USER root

# Common environment variables.
ENV PYTHONUNBUFFERED=1 \
    PYTHONIOENCODING=UTF-8 \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    PIP_NO_CACHE_DIR=off \
    BASH_ENV=/opt/workshop/etc/profile \
    ENV=/opt/workshop/etc/profile \
    PROMPT_COMMAND=". /opt/workshop/etc/profile" \
    BUILDAH_ISOLATION=chroot \
    _BUILDAH_STARTED_IN_USERNS=""

COPY workshop /opt/workshop
COPY s2i/. /usr/libexec/s2i/

    # Install additional common utilities.
RUN HOME=/root && \
    INSTALL_PKGS="sudo java-1.8.0-openjdk-devel maven skopeo" && \
    yum -y --setopt=tsflags=nodocs install $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    pip install --no-cache-dir supervisor==4.2.0 mercurial==5.4.2 && \
    curl -s -L -o /usr/local/bin/jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 && \
    chmod +x /usr/local/bin/jq && \
    curl -sLo /usr/local/bin/yq https://github.com/mikefarah/yq/releases/download/3.3.4/yq_linux_amd64 && \
    chmod +x /usr/local/bin/yq && \
    yum -y update mod_http2 && \

    # Install OpenShift clients.
    declare -A ocp_kube_versions=( [3.11.250]=1.11 [4.3]=1.16 [4.4]=1.17 [4.5]=1.18 ) && \
    for oc_version in "${!ocp_kube_versions[@]}"; do \
        major_version=$(echo $oc_version | cut -d. -f1) && \
        minor_version=$(echo $oc_version | cut -d. -f2) && \
        { [ "$major_version" = 3 ] && dl_link="$oc_version" || dl_link="oc/$oc_version" ; } && \
        curl -s -o /tmp/oc.tar.gz https://mirror.openshift.com/pub/openshift-v${major_version}/clients/${dl_link}/linux/oc.tar.gz && \
        tar -C /usr/local/bin -zxf /tmp/oc.tar.gz oc && \
        mv /usr/local/bin/oc /usr/local/bin/oc-${major_version}.${minor_version} && \
        ln -s /usr/local/bin/oc-${major_version}.${minor_version} /usr/local/bin/kubectl-${ocp_kube_versions[$oc_version]} && \
        rm /tmp/oc.tar.gz ; \
    done && \
    curl -sL -o /usr/local/bin/odo https://mirror.openshift.com/pub/openshift-v4/clients/odo/latest/odo-linux-amd64 && \
    chmod +x /usr/local/bin/odo && \


    # Install Butterfly
    HOME=/opt/workshop/butterfly && \
    cd /opt/workshop/butterfly && \
    pip install --no-cache-dir virtualenv && \
    virtualenv /opt/workshop/butterfly && \
    source /opt/workshop/butterfly/bin/activate && \
    pip install --no-cache-dir -r requirements.txt && \
    /opt/workshop/butterfly/install-fonts.sh && \
    /opt/workshop/butterfly/fixup-styles.sh && \
    rm /opt/app-root/etc/scl_enable && \

    # Install gateway application using SCL Node.js 12.
    HOME=/opt/workshop/gateway && \
    cd /opt/workshop/gateway && \
    yum module reset -y nodejs && \
    yum module install -y nodejs:12 && \
    npm install --production && \
    chown -R 1001:0 /opt/workshop/gateway/node_modules && \
    fix-permissions /opt/workshop/gateway/node_modules && \

    # Install mod_wsgi-express for starting Apache with WebDAV.
    HOME=/opt/workshop/webdav && \
    mkdir /opt/workshop/webdav && \
    cd /opt/workshop/gateway && \
    virtualenv /opt/workshop/webdav && \
    source /opt/workshop/webdav/bin/activate && \
    pip install --no-cache-dir mod_wsgi && \

    # Adjust permissions
    cp /opt/workshop/bin/start-singleuser.sh /opt/app-root/bin/ && \
    echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%wheel/# %wheel/' /etc/sudoers && \
    chmod g+w /etc/passwd && \

    # Install and Configure buildah
    yum -y copr enable rhcontainerbot/container-selinux && \
    wget -P /etc/yum.repos.d https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo && \
    yum -y install buildah fuse-overlayfs && \
    sed -i -e 's|^#mount_program|mount_program|g' -e '/additionalimage.*/a "/var/lib/shared",' -e 's|"overlay"|"vfs"|g' /etc/containers/storage.conf && \
    sed -i.bak \
        -e "/\[registries.search\]/{N;s/registries = \[.*\]/registries = ['docker.io', 'registry.fedoraproject.org', 'quay.io', 'registry.centos.org']/}" \
        -e "/\[registries.insecure\]/{N;s/registries = \[.*\]/registries = ['docker-registry.default.svc:5000','image-registry.openshift-image-registry.svc:5000']/}" \
        /etc/containers/registries.conf

COPY containers/libpod.conf /etc/containers/

COPY containers/sudoers.d/ /etc/sudoers.d/


RUN mkdir -p /opt/app-root/etc/init.d && \
    mkdir -p /opt/app-root/etc/profile.d && \
    mkdir -p /opt/app-root/etc/supervisor && \
    mkdir -p /opt/app-root/gateway/routes && \
    chown -R 1001:0 /opt/app-root && \
    fix-permissions /opt/app-root

COPY .bash_profile /opt/app-root/src/.bash_profile

RUN virtualenv /opt/app-root && \
    source /opt/app-root/bin/activate && \
    pip install --no-cache-dir -U pip setuptools wheel && \
    pip install --no-cache-dir ansible==2.9.11 openshift==0.11.2 jmespath==0.10.0 && \
    ln -s /opt/workshop/bin/oc /opt/app-root/bin/oc && \
    ln -s /usr/local/bin/odo /opt/app-root/bin/odo && \
    ln -s /opt/workshop/bin/kubectl /opt/app-root/bin/kubectl && \
    chown -R 1001:0 /opt/app-root && \
    fix-permissions /opt/app-root -P && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache /var/log/dnf* /var/log/yum.*

COPY profiles/. /opt/workshop/etc/profile.d/

RUN ln -s /opt/workshop/etc/supervisord.conf /etc/supervisord.conf

LABEL io.k8s.display-name="Terminal" \
      io.openshift.expose-services="10080:http" \
      io.openshift.tags="builder,butterfly" \
      io.openshift.s2i.scripts-url=image:///usr/libexec/s2i

EXPOSE 10080

USER 1001

CMD [ "/usr/libexec/s2i/run" ]
