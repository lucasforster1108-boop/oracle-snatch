[中文](README.md) | **English**

# oracle-snatch

A small Python loop that keeps asking Oracle Cloud's official API for a free ARM instance (VM.Standard.A1.Flex) until one attempt happens to land when capacity opens up and the API says yes.

## Why I wrote this

The Always Free A1.Flex allowance (4 cores / 24 GB in total) is the best free server deal out there, period. The problem: in popular regions the console basically always says "Out of host capacity". That capacity is not gone forever — it gets released in small, irregular waves, and nobody notifies you. Refreshing the console for a week is a waste of a life. Let a script do the asking.

In October 2026 I ran a rough version of this loop in the Tokyo region for several days. The log was just "out of capacity" over and over, plus a few 429 rate limits when I set the interval too aggressively. Eventually it grabbed that 4-core / 24 GB machine. Most of my self-hosted services run on it now, and the bill is still 0 to this day. This repo is that loop, cleaned up, with all config pulled out into a file.

## What it does

- Uses only Oracle's official API (the `oci` Python SDK). No console automation, no browser simulation.
- Polls every Availability Domain (AD) in your region, one attempt per AD per round. Tokyo has a single AD; regions with more ADs give you better odds.
- Treats "out of capacity" as "not yet": waits a bit, moves to the next AD, starts the next round.
- Backs off long (300 seconds by default) on a 429. Ramming through a rate limit only makes it stick around longer.
- Stops on quota errors (LimitExceeded) and tells you why: your free allowance is used up, and retrying ten thousand more times will not change that.
- Once it wins, waits for the instance to reach RUNNING, grabs the public IP, writes it to `snatched.json`, optionally sends a Telegram notification, and exits.
- `--dry-run` only validates your setup: lists the ADs and confirms which image it would use, without launching anything. Run this first.

What it does not do: it will not register an account for you, build a VCN, generate API keys, or conjure capacity that does not exist. It just asks, tirelessly, on your behalf.

## Install and use

You need an Oracle Cloud account and an OCI config file (`~/.oci/config`, with an API key set up). Anyone who has used the OCI CLI or SDK already has this file.

You also need a subnet OCID: in the console, Networking -> Virtual Cloud Networks -> your VCN -> Subnets. Pick a public subnet if you want a public IP (you do).

```bash
git clone https://github.com/lucasforster1108-boop/oracle-snatch.git
cd oracle-snatch
pip install -r requirements.txt

cp config.example.env config.env
# Edit config.env: the only required fields are SUBNET_ID and SSH_PUBLIC_KEY_FILE

python3 snatch.py --dry-run
python3 snatch.py
```

## Running it long-term

This is meant to run unattended for days, so put it on any always-on machine you already have (a free VPS somewhere else, a home server, a Raspberry Pi). A sample systemd service is included:

```bash
sudo cp oracle-snatch.service /etc/systemd/system/
sudo systemctl enable --now oracle-snatch
tail -f snatch.log
```

## Lessons learned the hard way

- The free A1 allowance is a total pool, not per-instance: 4 cores + 24 GB altogether. One 4/24 machine, or split it into two 2/12 machines. Go over and you get LimitExceeded — the script stops on purpose, do not blame it.
- Upgrading the account to PAYG (pay-as-you-go) does not eat your free allowance. A1 usage inside the allowance is still free, and capacity honestly feels a bit easier to get. My account is PAYG, the bill is still 0, and I keep a budget alert as a safety net.
- Boot volumes count toward the 200 GB free block storage total. 50 GB is plenty for a small server; do not max out 200 just because it is free — the leftover allowance is more useful for a second machine.
- Capacity drops are irregular, often in the middle of the night. That is the whole point of the loop: at 4 a.m. it is awake, you do not have to be.
- Do not shorten the retry interval to a few seconds. You will not snatch anything faster; you will just eat 429s and then get nothing for a while.

## Files

- `snatch.py`: all the logic, one file
- `config.example.env`: every config option, commented
- `oracle-snatch.service`: sample systemd service

## License

MIT
