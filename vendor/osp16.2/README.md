## Applying the arbiterd patch to nova-compute image

While the TCIB process isn't yet supported and documented in Red Hat Openstack Platform 16.x, it's possible to modify images using the
`tripleo-modify-image` role during the `tripleo container image prepare` phase of a deployment, as described [here](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.2/html/transitioning_to_containerized_services/obtaining-container-images#modifying-container-images-with-a-custom-dockerfile).

We've provided a sample configuration file based on upstream deployments but that should be similar downstrean.

### Procedure

1. From the stack home, clone the [arbiterd repo](https://github.com/SeanMooney/arbiterd):
    ```
    [stack@undercloud-0 ~]$ git clone https://github.com/SeanMooney/arbiterd.git
    ```

2. Modify the `containers-prepare-parameter.yaml` to exclude the `nova-compute` container from the default config and including it in a new config section. Here's a difference of the changes that need to be included:
    ```
    diff containers-prepare-parameter.yaml containers-prepare-parameter-sample.yaml
    26a27,45
    >     excludes:
    >       - "nova-compute:"
    >   - push_destination: true
    >     includes:
    >       - "nova-compute:"
    >     set:
    >       name_prefix: openstack-
    >       name_suffix: ''
    >       rhel_containers: false
    >       tag: 16.2
    >       namespace: my-container.registry.domain.com:5002/rh-osbs
    >     modify_role: tripleo-modify-image
    >     modify_append_tag: "-arbiterd"
    >     modify_vars:
    >       tasks_from: modify_image.yml
    >       modify_dir_path: /home/stack/arbiterd/vendor/osp16.2/
    ```

Make sure to update the config's `set` section based on your deployment, using the same namespace/tag/prefix/suffix as the default config section.

4. Copy the repository files to the docker build directory:
    ```
    [stack@undercloud-0 ~]$ cp /etc/yum.repos.d/*.repo /home/stack/arbiterd/vendor/osp16.2/
    ```

3. Rebuild the containers
    ```
    [stack@undercloud-0 ~]$ sudo openstack tripleo container image prepare -e /home/stack/containers-prepare-parameter.yaml --output-env-file /home/stack/docker-images.yaml
    ```

4. Proceed with a minor stack update as described in our [documentation](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.2/html/keeping_red_hat_openstack_platform_updated/index) or simply re-run the deployment command.

5. Build arbiterd image
    ```
    [stack@undercloud-0 arbiterd]$ sudo buildah bud -f Dockerfile -t undercloud-0.ctlplane.redhat.local:8787/rh-osbs/rhosp16-arbiterd:16.2_20211110.2
    ```

6. Upload it to the registry
    ```
    [stack@undercloud-0 arbiterd]$ sudo openstack tripleo container image push \
    --local --registry-url undercloud-0.ctlplane.redhat.local:8787 \
    undercloud-0.ctlplane.redhat.local:8787/rh-osbs/rhosp16-arbiterd:16.2_20211110.2
    ```
