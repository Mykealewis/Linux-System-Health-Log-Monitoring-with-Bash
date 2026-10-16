# Linux-System-Health-Log-Monitoring-with-Bash
Linux System Health &amp; Log Monitor  A lightweight Bash-based monitoring tool that collects essential Linux system metrics and inspects system logs to help identify potential performance issues and service failures.

The script monitors CPU utilization, system load, memory usage, disk space, and resource-intensive processes. It also reviews recent system log warnings and errors, checks for failed systemd services, and generates a timestamped health report for troubleshooting and routine system maintenance. 

<h1>Features</h1>

CPU Monitoring: Estimates CPU utilization and displays system load averages.

Memory Analysis: Reports memory usage and flags high utilization based on configurable thresholds.

Disk Monitoring: Checks mounted filesystem usage and highlights disks exceeding the configured limit.

Process Inspection: Identifies processes with the highest CPU and memory consumption.

Log Analysis: Reviews recent system journal entries or available traditional log files for warnings, errors, and critical events.

Service Health Checks: Identifies failed systemd services.

Automated Reporting: Saves system health findings to a timestamped report in /tmp.

Configurable Thresholds: Allows users to customize memory and disk usage warning levels through environment variables.

<h1>Technologies</h1>

Bash / Shell Scripting

-Linux System Administration

-Linux System Logs (journalctl, /var/log)

-Process and Resource Monitoring

-Systemd Service Management

<h1>Purpose</h1>

This project demonstrates practical Linux administration, Bash automation, system troubleshooting, and log analysis. It aims to simplify routine health checks and provide a repeatable starting point for investigating potential system issues.

<h1>Future Improvements</h1>

Scheduled execution with cron or systemd timers

Automated alerts for resource threshold violations

Historical metric tracking and trend analysis

Structured logging and improved error handling

Integration with centralized monitoring solutions
