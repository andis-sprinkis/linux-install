#!/usr/bin/env bash

# Pad echo output with line breaks

echo_br_bottom() {
  echo $1; echo
}

echo_br_top() {
  echo; echo $1
}

echo_br_both() {
  echo; echo $1; echo
}

# Annotate functions and stages of scripts

fn_call() {
  for fn in "$@"; do
    echo_br_bottom "Function $fn has started..."
    $fn
    echo_br_bottom "Function $fn has completed!"
  done
}

echo_stage_status() {
  if [ "$1" -eq "0" ]; then
    echo_br_bottom "Starting $2 stage..."
  fi 

  if [ "$1" -eq "1" ]; then
    echo_br_bottom "$2 stage has completed!"
  fi 
}

# Read concealed password input
# Credit: Stéphane Chazelas - https://unix.stackexchange.com/a/223000

read_pw_input() {
  PW_INPUT="$(
    # always read from the tty even when redirected:
    exec < /dev/tty || exit # || exit only needed for bash

    # save current tty settings:
    tty_settings=$(stty -g) || exit

    # schedule restore of the settings on exit of that subshell
    # or on receiving SIGINT or SIGTERM:
    trap 'stty "$tty_settings"' EXIT INT TERM

    # disable terminal local echo
    stty -echo || exit

    # prompt on tty
    printf "Password: " > /dev/tty

    # read password as one line, record exit status
    IFS= read -r password; ret=$?

    # display a newline to visually acknowledge the entered password
    echo > /dev/tty

    # return the password for $PW_INPUT
    printf '%s\n' "$password"
    exit "$ret"
  )"
}

# Prompt with user error handling for concealed password input

confirm_pw_input() {
  prefix_usr_input_err="User input error -"
  read_pw_input && password_1=$PW_INPUT

  if [ -z "$password_1" ]; then
    password_1="" password_2="" PW_INPUT=""
    echo_br_top "$prefix_usr_input_err password must not be empty."
    confirm_pw_input $1 && return 0; return 1
  fi

  echo_br_top "Repeat the password."
  read_pw_input && password_2=$PW_INPUT

  if [ -z "$password_2" ]; then
    password_1="" password_2="" PW_INPUT=""
    echo_br_top "$prefix_usr_input_err password must not be empty."
    confirm_pw_input $1 && return 0; return 1
  fi

  if [ ! "$password_1" == "$password_2" ]; then
    password_1="" password_2="" PW_INPUT=""
    echo_br_top "$prefix_usr_input_err passwords must match."
    confirm_pw_input $1 && return 0; return 1
  fi

  password_1="" password_2="" PW_INPUT=""
  return 0
}

# Prompt based on confirm_pw_input to update user password

set_user_pw_confirm() {
  echo "Set the password for user $1."

  confirm_pw_input
  (echo "$PW_INPUT"; echo "$PW_INPUT") | passwd $1
  PW_INPUT=""

  echo_br_both "Password for user $1 has been set."
}
