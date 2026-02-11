# Jamf_zero_touch_scripts ? Adobe_Nuke_Apps
Scripts, utilities, and workflows for Jamf  Pro / ABM auto enrollment
Adobe__CC25
#!/bin/bash

## Script to install Adobe CC2025 applications using Swift Dialog
## Blocks user interaction, keeps Mac awake, and shows progress with real-time installer output.

# --- Configuration ---
SWIFT_DIALOG_PATH="/usr/local/bin/dialog" # Path to Swift Dialog
INSTALL_PACKAGE="/Library/Application Support/JAMF/Waiting Room /UPF-R26-Adobe-BaselineCC25-727_en_US_MACARM.pkg" # Path to your installer package
DIALOG_ICON="/private/var/tmp/mando_icon.png " # Icon for the dialog
DIALOG_TITLE="Adobe Creative Cloud 2025 Installation"
DIALOG_MESSAGE="Please wait while Adobe Creative Cloud 2025 applications are being installed. This may take some time. Your Mac will stay awake during this process. Do not close your lid or power off your machine. \n\n**Installation Log:**"
INSTALLER_LOG="/var/log/adobe_cc25_install_$(date +%Y%m%d%H%M%S).log" # Unique log file for each run

# --- Functions ---

# Function to display Swift Dialog and capture its PID
start_dialog() {
    "${SWIFT_DIALOG_PATH}" \
        --title "${DIALOG_TITLE}" \
        --message "${DIALOG_MESSAGE}" \
        --icon "${DIALOG_ICON}" \
        --progress 0 \
        --button1text "Installing..." \
        --button1disabled \
        --helpmessage "Please do not interrupt the installation." \
        --width 700 \
        --height 500 \
        --position center \
        --ontop \
        --movable \
        --progressRing \
        --progressText "Initializing installation..." \
        --list "Installer Output" \
        --liststyle "font-size: 10px; font-family: monospace;" \
        --textbox "Starting installation..." \
        --textbox-progress \
        --minibox \
        --commandfile "/var/tmp/dialog_command.log" & # Command file for real-time updates
    DIALOG_PID=$!
    echo "Swift Dialog PID: ${DIALOG_PID}"
}

# Function to update dialog progress text and list (Installer output)
update_dialog() {
    local command_string=$1
    echo "${command_string}" >> "/var/tmp/dialog_command.log"
}

# Function to keep the Mac awake
caffeinate_on() {
    echo "Keeping Mac awake..."
    caffeinate -dimsu &
    CAFFEINATE_PID=$!
}

# Function to stop caffeinate
caffeinate_off() {
    if [[ -n "${CAFFEINATE_PID}" ]]; then
        echo "Stopping caffeinate (PID: ${CAFFEINATE_PID})..."
        kill "${CAFFEINATE_PID}" 2>/dev/null
        unset CAFFEINATE_PID
    fi
}

# Function to kill Swift Dialog gracefully, then forcefully if needed
kill_dialog() {
    if [[ -n "${DIALOG_PID}" ]]; then
        echo "Attempting to kill Swift Dialog (PID: ${DIALOG_PID})..."
        kill "${DIALOG_PID}" 2>/dev/null
        # Give it a moment to quit gracefully
        sleep 1
        # Check if it's still running
        if kill -0 "${DIALOG_PID}" 2>/dev/null; then
            echo "Swift Dialog (PID: ${DIALOG_PID}) still running, forcing kill..."
            kill -9 "${DIALOG_PID}" 2>/dev/null
        fi
        wait "${DIALOG_PID}" 2>/dev/null
        unset DIALOG_PID
    fi
    # Clean up the command file
    rm -f "/var/tmp/dialog_command.log"
}

# --- Main Script ---

# Check if Swift Dialog is installed
if [[ ! -x "${SWIFT_DIALOG_PATH}" ]]; then
    echo "Error: Swift Dialog not found at ${SWIFT_DIALOG_PATH}. Please install it."
    exit 1
fi

# Check if the installation package exists
if [[ ! -f "${INSTALL_PACKAGE}" ]]; then
    echo "Error: Installation package not found at ${INSTALL_PACKAGE}."
    exit 1
fi

# Ensure the log file is clean before starting
rm -f "${INSTALLER_LOG}"

# Trap to ensure dialog is killed and caffeinate is stopped on exit or error
trap 'kill_dialog; caffeinate_off; exit' SIGINT SIGTERM EXIT

# Start caffeinate
caffeinate_on

# Start Swift Dialog
start_dialog

# Give dialog a moment to initialize
sleep 2

# Start tailing the installer log and sending output to dialog's list
# This runs in the background
tail -f "${INSTALLER_LOG}" | while IFS= read -r line; do
    update_dialog "list: add: ${line}"
done &
TAIL_PID=$!
echo "Tailing log file (PID: ${TAIL_PID})..."

# Initial dialog update for starting installation
update_dialog "progresstext: Preparing installation..."
update_dialog "list: add: --- Starting Adobe CC2025 Installation ---"
update_dialog "list: add: Package: ${INSTALL_PACKAGE}"
update_dialog "list: add: Log File: ${INSTALLER_LOG}"
update_dialog "list: add: "

# Perform the installation
echo "Starting package installation: ${INSTALL_PACKAGE}"
pkg_install_command="sudo installer -pkg \"${INSTALL_PACKAGE}\" -target /"

# Update dialog and execute the actual package installation
update_dialog "progresstext: Executing Adobe CC2025 package installation..."
update_dialog "progress: 25" # Initial progress
update_dialog "list: add: Executing actual package installation command..."
update_dialog "list: add: This step may take a significant amount of time."

# Execute the installer command, redirecting stdout and stderr to our log file
eval "${pkg_install_command}" >> "${INSTALLER_LOG}" 2>&1
INSTALL_STATUS=$?

# Stop the tail process now that the installation is complete
if [[ -n "${TAIL_PID}" ]]; then
    echo "Stopping tail process (PID: ${TAIL_PID})..."
    kill "${TAIL_PID}" 2>/dev/null
    wait "${TAIL_PID}" 2>/dev/null
fi

if [[ ${INSTALL_STATUS} -eq 0 ]]; then
    echo "Adobe CC2025 applications package installed successfully."
    update_dialog "progress: 100"
    update_dialog "progresstext: Installation Complete!"
    update_dialog "button1text: Done"
    update_dialog "button1enabled: true" # Enable the "Done" button
    update_dialog "message: Adobe Creative Cloud 2025 applications package installed successfully. Your applications are ready to use. You may now click 'Done' to close this window."
    sleep 5 # Keep success message visible for a moment
else
    echo "Error: Adobe CC2025 applications package installation failed. Check ${INSTALLER_LOG} for details."
    update_dialog "progress: 100"
    update_dialog "progresstext: Installation Failed!"
    update_dialog "button1text: Close"
    update_dialog "button1enabled: true" # Enable the "Close" button
    update_dialog "message: **ERROR:** Adobe Creative Cloud 2025 applications package installation failed. Please contact IT support and provide the log file located at: \n\`${INSTALLER_LOG}\`"
    sleep 10 # Keep error message visible for longer
fi

# Wait for the user to click the "Done" or "Close" button
# Swift Dialog automatically closes when a button is clicked, or if commanded to quit.
# We keep the dialog alive by not killing it immediately after success/failure messages
# until the user clicks the button.

# If the script exits before the user clicks, the trap will handle killing the dialog.
# However, to explicitly wait for a button click, you'd typically remove the trap's exit,
# and use a `dialog` feature like `--json` output to detect the button press.
# For simplicity, and since the trap already covers cleanup, we'll let the user's interaction
# with the enabled button implicitly close the dialog, or the trap will catch script exit.

exit ${INSTALL_STATUS}
