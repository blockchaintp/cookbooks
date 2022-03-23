# SUSE Rancher Cookbook

Recipe for installing the Sextant Community Edition on a SUSE Rancher managed
Kubernetes cluster to deploy and manage blockchain networks.

## Prerequisites

* [SUSE Rancher](https://www.suse.com/products/suse-rancher/) v2.6 or later with
a Kubernetes cluster v1.19 or later
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) configured to
access your cluster

## Sign up for an evaluation licence

To install the Sextant Community edition, you will need user credentials
supplied by BTP. If you don't have these, you can request them
[here](https://www.blockchaintp.com/sextant/suse-rancher).

### License

Use of the Sextant Community Edition is governed by our
[Marketplace EULA](https://sextant-resources.s3.amazonaws.com/agreements/Blockchain+Technology+Partners+Limited+(Marketplace)+End+User+License+Agreement.pdf)
with the exception of Daml support which is subject to our
[Evaluation EULA](https://sextant-resources.s3.amazonaws.com/agreements/Blockchain+Technology+Partners+Limited+(Evaluation)+End+User+License+Agreement.pdf).

### Useful links

* [Sextant Overview](https://www.blockchaintp.com/sextant)
* [Sextant Docs](https://docs.blockchaintp.com/en/latest/sextant/overview/)

## Install Sextant

Log in to Rancher and select the cluster you want to install Sextant on,
in our example, this will be the local Rancher cluster.

![Local Rancher cluster](../images/rancher/local-cluster.png)

From the left menu, select _Apps & Marketplace_ and then _Charts_.
Choose the Sextant chart from the list of partner charts.

![Partner Charts](../images/rancher/partner-charts.png)

This will take you to the following screen.

![Namespace and Name](../images/rancher/install-metadata.png)

Here you need to specify the _namespace_ and _name_ for your Sextant
installatation. In this example we elect to use `sextant` in both cases.

!!!Note
    If the namespace doesn't exist the installation will automatically create
    this.

Now click the _Next_ button on the bottom right of the page.

!!!Important
    Make sure you have your BTP supplied credentials ready. As noted above,
    you can request these
    [here](https://www.blockchaintp.com/sextant/suse-rancher).

![User Credentials](../images/rancher/user-credentials.png)

On this screen you can configure your Sextant installation. On the lefthand side
there are three options:

* **User Credentials** - The only required fields are the `Username` and
`Password` credentials you obtained from BTP. These are entered here.
* **Ingress Settings** - If you'd like to enable an ingress for Sextant you can
specify this here. This is optional.
* **Database Settings** - If you'd like to use an external Postgres database,
you can specify this here. This is also optional.

Enter your user credentials in the form, and then click the _Install_ button
on the bottom right of the page.

Rancher will now install Sextant on your local cluster, it may take a few
minutes for the Sextant images to be pulled down to your cluster from our
private repo.

![Installing Sextant](../images/rancher/installing-sextant.png)

Once the installation has completed, you will see the NOTES from the
installation. In our example these are:

```
NOTES:
1. Get the initial Sextant application username and password by running this
command kubectl describe pod/sextant-0 --namespace sextant | grep INITIAL_
2. Get the application URL by running these commands:
export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=sextant" -o jsonpath="{.items[0].metadata.name}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl port-forward $POD_NAME 8080:80
```

Make a note of these instructions as we will now switch to a local terminal
window to finish setting up Sextant.

Once you've opened a local terminal, start by confirming that you can connect to
your Kubernetes cluster using `kubectl` by running this command:

```
kubectl get pods
```

![Check kubectl](../images/rancher/check-kubectl.png)

Then run the first command from the installation NOTES. In our example this is:

```json
kubectl describe pod/sextant-0 --namespace sextant | grep INITIAL_
```

This will display the initial user/password combination for the your Sextant
installation.

!!!Important
    Make sure you save this password, as it will not be possible to retrieve
    it if the Sextant deployment is restarted.

![Initial Password](../images/rancher/initial-password.png)

Now run the second command from the install notes. In our example this is:

```
export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=sextant" -o jsonpath="{.items[0].metadata.name}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl port-forward $POD_NAME 8080:80
```

This will set up a port forward to your Sextant install, and make it accessible
on your local machine at [http://127.0.0.1:8080](http://127.0.0.1:8080)

![Port forward](../images/rancher/port-forward.png)

Switch back to your browser and open the URL show in the terminal output. In
our case [http://127.0.0.1:8080](http://127.0.0.1:8080)

This will load the Sextant UI and you can log in using the initial password
retrieved earlier

![Sextant UI](../images/rancher/sextant-ui.png)

At this point you are all set to start using Sextant to deploy and manage
blockchain networks.

The first thing you will need to do is add a cluster to Sextant. Detailed
instructions on how to do this can be found
[here](https://docs.blockchaintp.com/en/latest/sextant/clusters/management/).

!!!Note
    Assuming that your local cluster has at least four nodes you can add this to
    Sextant.

!!!Important
    To access all Sextant's features you can also apply for a Sextant
    Enterprise Edition evaluation
    [here](https://www.blockchaintp.com/sextant/evaluation).
