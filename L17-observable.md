# Understanding Probes

- The probe itself is a simple test that is defined as a container property, which is often a command.

- Probes are used to test if the application that uses it is still functional.

- If the probe doesn't respond, the application is restarted.

- The following probe test types are defined in pods.spec.container:

  - exec: a command is executed and returns a zero exit value.

  - httpGet: an HTTP request returns a response code between 200 and 399.

  - tcpSocket: connectivity to a TCP socket (available port) is successful.

- Probes can be configured with a failureTreshold to determine how long it can take the application to react.

# Probe Types

- Kubernetes can use 3 different probe types:

  - livenessProbe: checks if the application is alive. Container will be restarted if the probe test fails.

  - readinessProbe: checks if the application is ready to service requests. Container will be removed from the list of available services if it fails.

  - startupProbe: used to verify initial startup of the application, useful if startup can be slow. No other probes are used before this probe finishes successfully.