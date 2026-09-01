# Grand Larceny Auto II — TryHackMe Write-up

- **Platform:** TryHackMe
- **Difficulty:** Medium
- **Category:** API Security / Reverse Engineering / Parameter Tampering
- **Architecture:** Godot Client + Remote HTTP API

## 1. Overview

Grand Larceny Auto II changes the approach used in the first challenge.

This time, the flag is not stored directly in the game. The challenge description states that the player must understand how the game communicates with the city's back office and prove that the heist was successfully completed.

The objective was therefore to understand the remote API workflow, reproduce the expected heist sequence, and identify a weakness in the final claim validation.

## 2. Connecting to the Backend

The challenge backend was accessible through the hostname `gla2.thm`.

I first mapped the current TryHackMe machine IP to the hostname using `/etc/hosts`:

```bash
echo "<MACHINE_IP> gla2.thm" | sudo tee -a /etc/hosts
```

After noticing that an older IP was still associated with the hostname, I removed the outdated entry and added the current machine IP:

```bash
sudo sed -i '/gla2.thm/d' /etc/hosts
echo "<MACHINE_IP> gla2.thm" | sudo tee -a /etc/hosts
```

I then verified connectivity with:

```bash
ping -c 3 gla2.thm
```

<img width="772" height="248" alt="image" src="https://github.com/user-attachments/assets/2731e645-9017-43f9-b248-82a2e4ebd05a" />

The host responded successfully, confirming that the backend was reachable.

## 3. Understanding the API Workflow

The heist uses a stateful backend workflow.

A session is created first, and the server returns a randomized stash collection order.

In my run, the returned order was:

```text
[1, 2, 0]
```

The required checkpoint sequence was then:

```text
heat5
stash1
stash2
stash0
vault
```

Each successful checkpoint returned a new token that was required to continue the sequence.

The backend also enforced a timing requirement between checkpoint transitions, meaning that the requests could not simply be sent immediately one after another.

## 4. Claim Validation Flaw

During the analysis, I identified that the final claim included a privileged `role` value.

The role was derived from the completed checkpoint sequence using SHA-1.

For my completed route, the generated staff role was:

```text
179c38f0bb7c1736088aa30dca07e76b16b5e7dd
```

The important weakness was that the `role` field was not protected by the same integrity validation used for the other claim data.

This meant that a privileged role could be supplied without invalidating the request signature.

In practice, this created a parameter-tampering and signature-scope issue in the final `/claim` request.

## 5. Exploitation

I used a Python script to automate the complete workflow:

1. Create a new API session.
2. Retrieve the randomized `stash_order`.
3. Follow the required checkpoint sequence.
4. Respect the backend timing requirement.
5. Derive the staff role using SHA-1.
6. Submit the final privileged `/claim` request.

The script successfully completed the full sequence:

<img width="863" height="375" alt="image" src="https://github.com/user-attachments/assets/a61a6eee-a678-47f9-9ed1-8e3c6bf3794b" />

The backend accepted the final claim and returned:

```json
{"flag":"THM{Th4ts_th3_wr0ng_g4m3_t0mmy}"}
```

## 6. Flag

```text
THM{Th4ts_th3_wr0ng_g4m3_t0mmy}
```

## 7. What I Learned

Grand Larceny Auto II demonstrated the difference between client-side and server-side security controls.

Moving sensitive validation to a remote backend is stronger than relying entirely on client-side logic, but the server still needs to correctly protect every security-relevant parameter.

The main lesson from this challenge was that a cryptographic integrity mechanism only protects the values actually included in its validation scope. If a privilege-related parameter such as `role` is excluded, that field may still be modified without invalidating the request.

The challenge also reinforced the importance of understanding application state, request sequencing, and timing requirements when analyzing stateful APIs.
