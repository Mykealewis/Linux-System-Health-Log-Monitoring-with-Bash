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

<h1>Code</h1>


#!/usr/bin/env bash

# system_health_check.sh
# Collect Linux system metrics and inspect recent system logs.
# Read-only monitoring; writes a report to /tmp.

set -uo pipefail
umask 077

TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
REPORT="/tmp/system_health_report_${TIMESTAMP}.txt"
LOG_WINDOW="1 hour ago"

# Thresholds can be overridden through environment variables.
DISK_WARN=${DISK_WARN:-80}
MEM_WARN=${MEM_WARN:-85}

# Write to both the terminal and the report.
exec > >(tee -a "$REPORT") 2>&1

section() {
    printf '\n\n========== %s ==========\n' "$1"
}

section "SYSTEM HEALTH REPORT"
echo "Hostname: $(hostname)"
echo "Date: $(date)"
echo "User: $(id -un)"
echo "Kernel: $(uname -sr)"
echo "Report: $REPORT"

section "UPTIME AND LOAD"
uptime || true

if [[ -r /proc/loadavg ]]; then
    printf 'Load averages: '
    awk '{print $1, $2, $3}' /proc/loadavg
fi

section "CPU UTILIZATION"

# Calculate approximate CPU utilization from /proc/stat.
if [[ -r /proc/stat ]]; then
    read -r -a cpu1 < <(awk '/^cpu / {print $2,$3,$4,$5,$6,$7,$8}' /proc/stat)
    sleep 1
    read -r -a cpu2 < <(awk '/^cpu / {print $2,$3,$4,$5,$6,$7,$8}' /proc/stat)

    total1=0
    total2=0
    idle1=$((cpu1[3] + cpu1[4]))
    idle2=$((cpu2[3] + cpu2[4]))

    for value in "${cpu1[@]}"; do
        ((total1 += value))
    done
    for value in "${cpu2[@]}"; do
        ((total2 += value))
    done

    total_delta=$((total2 - total1))
    idle_delta=$((idle2 - idle1))

    if (( total_delta > 0 )); then
        awk -v total="$total_delta" -v idle="$idle_delta" \
            'BEGIN {
                printf "Approximate CPU utilization: %.1f%%\n",
                       100 * (total - idle) / total
            }'
    else
        echo "CPU utilization unavailable."
    fi
else
    echo "CPU statistics unavailable."
fi

section "MEMORY UTILIZATION"

if command -v free >/dev/null 2>&1; then
    free -h
    free -b | awk '/^Mem:/ {
        if ($2 > 0) {
            printf "Memory used (excluding reclaimable cache): %.1f%%\n",
                   100 * ($2 - $7) / $2
        }
    }'
else
    echo "'free' command unavailable."
fi

section "DISK UTILIZATION"
df -hP -x tmpfs -x devtmpfs |
    awk 'NR == 1 {print; next}
         {
             print
             usage = $5
             gsub(/%/, "", usage)
             if (usage + 0 >= warn)
                 printf "WARNING: %s is %s full (%s)\n",
                        $6, $5, $1
         }' warn="$DISK_WARN"

section "TOP PROCESSES BY CPU"
if command -v ps >/dev/null 2>&1; then
    ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 11
fi

section "TOP PROCESSES BY MEMORY"
ps -eo pid,comm,%cpu,%mem --sort=-%mem | head -n 11

section "RECENT SYSTEM LOG WARNINGS AND ERRORS"

if command -v journalctl >/dev/null 2>&1; then
    echo "Reading journal entries from the last $LOG_WINDOW."
    if ! journalctl --since "$LOG_WINDOW" -p warning \
        --no-pager -n 200 2>&1; then
        echo "Journal access failed or is restricted."
    fi
else
    echo "journalctl unavailable; checking traditional log files."

    found_log=false
    for logfile in /var/log/syslog /var/log/messages; do
        if [[ -r "$logfile" ]]; then
            found_log=true
            echo "--- $logfile (last 500 lines; filtered) ---"
            tail -n 500 "$logfile" |
                grep -iE 'warning|error|critical|failed|failure|panic' |
                tail -n 100 || true
        fi
    done

    if [[ "$found_log" == false ]]; then
        echo "No readable traditional system log files found."
    fi
fi

section "FAILED SYSTEMD SERVICES"

if command -v systemctl >/dev/null 2>&1; then
    systemctl --failed --no-pager 2>&1 ||
        echo "Could not retrieve failed services."
else
    echo "systemctl unavailable."
fi

section "THRESHOLD SUMMARY"

if command -v free >/dev/null 2>&1; then
    mem_used=$(free | awk '/^Mem:/ {
        if ($2 > 0) printf "%.0f", 100 * ($2 - $7) / $2
    }')
    if [[ -n "${mem_used:-}" ]] && (( mem_used >= MEM_WARN )); then
        echo "WARNING: Memory utilization is ${mem_used}%."
    else
        echo "Memory utilization is below the ${MEM_WARN}% threshold."
    fi
fi

echo "Disk warning threshold: ${DISK_WARN}%"
echo "Memory warning threshold: ${MEM_WARN}%"

section "REPORT COMPLETE"
echo "Report saved to: $REPORT"
echo "Review warnings in context before taking corrective action."
