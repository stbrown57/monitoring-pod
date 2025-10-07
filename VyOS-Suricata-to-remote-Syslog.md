You configure VyOS Suricata to forward events to a Syslog server in two main steps: **configure Suricata to send logs to the local syslog service** and **configure the VyOS system logging to forward those local logs to your remote Syslog server**.

The key to sending Suricata events to syslog is using the `set service suricata log eve filetype 'syslog'` command.

-----

## 1\. Configure Suricata for Syslog Output

You must be in configuration mode for these steps.

1.  **Set the EVE log output type to `syslog`**: This tells Suricata to send its detailed JSON events (EVE logs) to the system's logging facility instead of a file. You will also specify which types of events you want to send. The common types are `alert` (for intrusion detection events) and `drop` (for intrusion prevention events).

    ```bash
    set service suricata log eve filetype 'syslog'
    set service suricata log eve type 'alert'
    set service suricata log eve type 'drop'
    ```

2.  **Commit the Suricata configuration**: This saves the changes and regenerates the Suricata configuration file (`/run/suricata/suricata.yaml`).

    ```bash
    commit
    ```

3.  **Update and restart Suricata**: You must run the update command to ensure Suricata processes the new configuration and restarts with the syslog output enabled.

    ```bash
    update suricata
    ```

-----

## 2\. Configure VyOS System Logging

Next, you need to configure VyOS to forward the logs it receives from the local Suricata service to your remote Syslog server.

1.  **Configure the remote Syslog host**: Replace `<syslog-server-ip>` with the IP address of your remote Syslog server. You should also specify the **facility** and **level** of the logs you want to send. Suricata alerts are often associated with the `user` or a `local` facility (like `local7`).

    ```bash
    set system syslog host <syslog-server-ip> facility all level 'notice'
    set system syslog host <syslog-server-ip> protocol 'udp'
    set system syslog host <syslog-server-ip> port '514'
    ```

      * **`<syslog-server-ip>`**: The IP address of your remote Syslog server.
      * **`facility`**: The system source of the log message. Setting it to `all` will include all local logs, including those from Suricata. You can try a more specific facility if needed (e.g., `local7`).
      * **`level`**: The severity level to send (`notice` is a good starting point to capture most alerts).
      * **`protocol`**: Use `udp` (default) or `tcp` (more reliable for large log messages, but usually requires a specific port like 6514).
      * **`port`**: The port your Syslog server is listening on (default is 514 for UDP/TCP).

2.  **Commit and Save**:

    ```bash
    commit
    save
    ```

Suricata events will now be sent to the local VyOS syslog service, and the VyOS system logging will forward those logs to your configured remote Syslog server.

-----

### 💡 Alternative (Advanced)

If you encounter issues with the built-in syslog forwarding, you can manually modify the Suricata EVE log configuration file:

1.  **SSH** into your VyOS router and switch to the root user.

2.  The active configuration file for Suricata is typically located at `/run/suricata/suricata.yaml`.

3.  Manually edit the `outputs` section for `eve-log` to look something like this, replacing the default `filetype: syslog` that the CLI sets:

    ```yaml
    # /run/suricata/suricata.yaml
    outputs:
      - eve-log:
          enabled: yes
          filetype: unix_dgram  # Use a Unix socket for high performance
          # Replace with desired log types
          types:
            - alert
            - drop
          # ... other options
          # Other outputs for logging may be present here (e.g., standard syslog)
    ```

    You would then need to configure the local VyOS Syslog daemon (rsyslog) to monitor the **Suricata EVE log file** or **Unix socket** (if you didn't set `filetype: syslog`) and forward its contents to the remote server. However, the VyOS CLI method in Section 1 is the intended way to achieve this.