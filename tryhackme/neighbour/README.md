# TryHackMe Neighbour CTF Write-up (IDOR)

**Author:** Rully Miftahur Rozaq

## Introduction

This report documents the steps used to solve the Neighbour CTF challenge on TryHackMe, which focuses on exploiting IDOR (Insecure Direct Object Reference). IDOR is a vulnerability that allows a user to manipulate an identifier to access another user's data.

---

## Walkthrough

### 1. Preparing and Accessing the Target

* First, start the TryHackMe machine to obtain the target IP address.
* Open the supplied IP address. In this case, the target at `10.48.173.31` displayed a login form.

### 2. Finding Guest Credentials

* With no account available, inspect the page source by pressing CTRL + U.
* The source code exposed guest account credentials. Log in using the username `guest` and password `guest`.

### 3. Exploiting IDOR

* After logging in as the guest, the website displayed a warning against viewing other users' profiles.
* Test for IDOR by changing the URL's user parameter from `guest` to `admin`.
  * **Original URL:** `http://10.48.173.31/profile.php?user=guest`
  * **Modified URL:** `http://10.48.173.31/profile.php?user=admin`

---

## Final Result

Changing the user parameter to `admin` opened the administrator's profile and revealed the required flag.

**Flag:**
> `flag{66be95c478473d91a5358f2440c7af1f}`
