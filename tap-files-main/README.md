# K8s Resources and Scripts to install and configure Carbon Black Image Scanner in TAP 1.3

## 1. Create a `Secret` containing your CBC credentials

    kubectl apply -f 01-secret.yaml

## 2. Install the `carbonblack` package

    tanzu package install carbonblack-scanner \
    --package-name carbonblack.scanning.apps.tanzu.vmware.com \
    --version 1.0.0-beta.2 \
    --namespace tap-install \
    --values-file 02-values.yaml

## 3. Create the `ScanPolicies` that are going to be used for Image Scan and Source Scan

    kubectl apply -f 03-scan-policies.yaml

## 4. Create a `Pipeline` resource

    kubectl apply -f 04-pipeline.yaml

## 5. Configure the Out-Of-The-Box Supply Chain to use `carbonblack` for image scanner

- retrieve your existing TAP configuration and store it in a file:

    ```shell
    kubectl get secret tap-values -n tap-install -o jsonpath="{.data['tap-values\.yaml']}"  | base64 -d > tap-values.yaml
    ```

- modify your newly created `tap-values.yaml` file by adding the followin values:

    ```yaml
    ootb_supply_chain_testing_scanning:
      scanning:
        image:
          template: carbonblack-private-image-scan-template
          policy: carbonblack-scan-policy
    ```

**NOTE:** The `ootb_supply_chain_testing_scanning` probably already exists, be careful to not delete the other keys under it.

## 6. Configure the Metadata Store

- create the resource required to make the Tap GUI communicate with the Metadata Store:

    ```shell
    kubectl apply -f 05-metadata-store-config.yaml
    ```

- get the Metadata Store Access Token:

    ```shell
    kubectl get secret $(kubectl get sa -n metadata-store metadata-store-read-client -o json | jq -r '.secrets[0].name') -n metadata-store -o json | jq -r '.data.token' | base64 -d
    ```

- add the proxy config to the `tap-values.yaml` file:

    ```yaml
    tap_gui:
      app_config:
        proxy:
          /metadata-store:
            target: https://metadata-store-app.metadata-store:8443/api/v1
            changeOrigin: true
            secure: false
            headers:
              Authorization: "Bearer <ACCESS-TOKEN>"
              X-Custom-Source: project-star
    ```

    where `ACCESS-TOKEN` is the token obtained with the previous command

- update your TAP install with the new `tap-values.yaml` file:

    ```shell
    tanzu package installed update tap -p tap.tanzu.vmware.com -v <TAP_VERSION>  --values-file tap-values.yaml -n tap-install
    ```

**NOTE:** The `TAP_VERSION` can be optained by listing your already existing packages:

    tanzu package available list -n tap-install | grep tap

**NOTE:** If this instructions stop working at some point, check the full docs [here](https://docs-staging.vmware.com/en/draft/VMware-Tanzu-Application-Platform/1.3/tap/GUID-tap-gui-plugins-scc-tap-gui.html#enable-cve-scan-results-2).

## 7. Verify the installation

### Via `ImageScan`

You can verify the installation by creating an `ImageScan`:

    kubectl apply -f 06-image-scan.yaml

and then seeing the results:

    kubectl get imagescan -n my-apps

**NOTE:** It takes a few minutes for the image scan to be completed.

### Via `Workload`

Or you can test the whole supply chain by creating a Workload:

    tanzu apps workload create tanzu-java-web-app \
      --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
      --git-branch main \
      --type web \
      --label apps.tanzu.vmware.com/has-tests=true \
      --label app.kubernetes.io/part-of=component-a \
      --namespace my-apps \
      --yes

and seeing the results:

    tanzu apps workload get tanzu-java-web-app -n my-apps

or

    kubectl get imagescan -n my-apps

**NOTE:** It takes ~10 minutes for the workload to be completed.

## BONUS: Getting access to the TAP GUI

To access the TAP GUI get the address of the HTTP proxy:

    $ kubectl get httpproxy -A
    NAMESPACE            NAME                                                              FQDN                                                          TLS SECRET   STATUS   STATUS DESCRIPTION
    tap-gui              tap-gui                                                           tap-gui.anton-testing.tapdemo.vmware.com                                    valid    Valid HTTPProxy

And open `http://tap-gui.anton-testing.tapdemo.vmware.com` in your browser.

**NOTE:** Make sure you are opening `http://`, not `https://`, because the latter would not resolve.

## BONUS: Workaround for missing `ServiceAccount` permission

If during the creation of a `Workload` you get the following error:

    message: 'unable to apply object [my-apps/tanzu-java-web-app] for resource [config-provider]
    in supply chain [source-test-scan-to-url]: failed to get unstructured [my-apps/tanzu-java-web-app]
    from api server: podintents.conventions.carto.run "tanzu-java-web-app" is forbidden:
    User "system:serviceaccount:my-apps:default" cannot get resource "podintents" in
    API group "conventions.carto.run" in the namespace "my-apps"'

fix this by editing the `Role` associated with this `ServiceAccount`:

    kubectl edit role -n my-apps default

and add this entry:

    - apiGroups:
      - conventions.carto.run
      resources:
      - podintents
      verbs:
      - '*'

to the `rules` property.

**NOTE:** This bug should be fixed by the TAP team in some of the next releases.
