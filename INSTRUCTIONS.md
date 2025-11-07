# Testing the EDC Management API with Postman and Kubernetes

This guide explains how to use the provided Postman collection (`EDC Management API.postman_collection.json`) to test the Eclipse Data Connector (EDC) Management API running in a local Kubernetes cluster.

## Prerequisites

1.  **Postman:** Ensure you have the [Postman desktop client](https://www.postman.com/downloads/) installed.
2.  **Local Kubernetes Cluster:** You need a running local Kubernetes cluster, such as [Minikube](https://minikube.sigs.k8s.io/docs/start/), [Kind](https://kind.sigs.k8s.io/), or the one included with Docker Desktop.
3.  **Deployed EDC:** The EDC connector must be deployed to your local Kubernetes cluster.
4.  **kubectl:** The `kubectl` command-line tool must be installed and configured to communicate with your local cluster.

## Step 1: Expose the Management API Service

To access the Management API from your local machine, you need to forward a local port to the service running in your Kubernetes cluster.

1.  **Find the Service Name and Port:**
    First, find the name of the Kubernetes service that exposes the management API. It will typically contain "control-plane", "management", or "edc" in its name.

    ```bash
    kubectl get services
    ```

    Look for the service that exposes the management API port (commonly `8181` or similar).

2.  **Forward the Port:**
    Once you've identified the service name and port, use the following command to forward a local port to it. Replace `YOUR_SERVICE_NAME` and `MANAGEMENT_PORT` with the correct values. We'll use local port `8181`.

    ```bash
    kubectl port-forward service/YOUR_SERVICE_NAME 8181:MANAGEMENT_PORT
    ```

    For example, if your service is named `edc-control-plane` and the management port is `8181`, the command would be:

    ```bash
    kubectl port-forward service/edc-control-plane 8181:8181
    ```

    Keep this command running in a terminal. You can now access the management API at `http://localhost:8181`.

## Step 2: Import the Postman Collection

1.  Open Postman.
2.  Click **File > Import...** (or `Ctrl+O` / `Cmd+O`).
3.  Drag and drop the `EDC Management API.postman_collection.json` file into the import window, or click **Upload Files** to select it.
4.  Click **Import**. The collection will now appear in the "Collections" tab on the left.

## Step 3: Configure the Postman Environment

To avoid hardcoding the API's address, we'll use a Postman environment.

1.  In the left pane, click on the **Environments** tab.
2.  Click the **+** button to create a new environment.
3.  Name the environment "EDC Local k8s".
4.  Add a new variable:
    *   **Variable:** `BASE_URL`
    *   **Initial Value:** `http://localhost:8181/api/management`

    *Note: The context path `/api/management` is a common default. If your deployment uses a different path for the management API, adjust the `BASE_URL` value accordingly.*

5.  Click **Save** (or `Ctrl+S` / `Cmd+S`).
6.  In the top-right corner of Postman, select the "EDC Local k8s" environment from the dropdown menu.

## Step 4: Configure Authentication

The EDC Management API requires authentication. The provided collection does not include authentication headers, as the specific method can vary. You will likely need to add an API key.

1.  In the collection, select a request you want to send.
2.  Go to the **Headers** tab.
3.  Add a new key-value pair for your API key. This is commonly an `X-Api-Key` header.
    *   **KEY:** `X-Api-Key`
    *   **VALUE:** `YourSuperSecretApiKey`

    *Note: For convenience, you can set this at the collection level. Click the three dots next to the collection name, select "Edit", and add the header under the "Headers" tab. This will apply the header to all requests in the collection.*

## Step 5: Send Requests

You are now ready to test the API.

1.  Expand the "EDC Management API" collection in the left pane.
2.  Select any request.
3.  Click the **Send** button.

You should receive a response from the EDC connector running in your Kubernetes cluster.
