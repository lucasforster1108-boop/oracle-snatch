[English](README.md) | [中文](README_zh.md)

# oracle-snatch

A small Python loop that keeps asking Oracle Cloud for a free-tier ARM instance (VM.Standard.A1.Flex) until capacity shows up and the API says yes.

## Why this exists

The Always Free A1.Flex tier (4 OCPU / 24 GB total) is the best free VPS deal anywhere, and in busy regions the console answer is almost always "Out of host capacity". Capacity is not gone forever, it comes back in bursts nobody announces. Clicking refresh in the console for a week is a waste of life, so I let a script do the asking instead.

I ran a rough version of this in October 2026 against the Tokyo region. The log was page after page of "out of capacity", with a 429 every now and then when I got impatient and set the interval too low. It landed the 4 OCPU / 24 GB box that now runs most of my self-hosted services, and it has cost exactly $0 since. This repo is that loop, cleaned up and made configurable.

## What it does

- Uses only the official OCI API (Python `oci` SDK). No console automation, no browser tricks.
- Rotates through every availability domain in your region, one launch attempt per AD per round. (Tokyo only has one AD, multi-AD regions get better odds.)
- "Out of capacity" is treated as "not yet": wait a bit, next AD, next round.
- HTTP 429 gets a long backoff (300 s by default). Pushing through a 429 just makes the throttling stick around.
- Quota errors (LimitExceeded) stop the script with a clear message. That error means your free A1 allowance is already used up, retrying will never help.
- On success it waits for the instance to reach RUNNING, grabs the public IP, writes `snatched.json`, optionally pings you on Telegram, and exits.
- `--dry-run` validates your config, lists the ADs and shows which image would be used, without launching anything. Run this first.

What it does not do: it does not create your account, your VCN, or your API keys, and it cannot magic capacity into existence. It just asks politely, forever.

## Setup

You need an Oracle Cloud account and the OCI CLI config file (`~/.oci/config`) with an API key. If you have ever used the OCI CLI or the SDK, you already have this.

You also need one subnet OCID: in the console, Networking -> Virtual Cloud Networks -> your VCN -> Subnets. Pick a public subnet if you want a public IP (you do).

```bash
git clone https://github.com/lucasforster1108-boop/oracle-snatch.git
cd oracle-snatch
pip install -r requirements.txt

cp config.example.env config.env
# edit config.env: SUBNET_ID and SSH_PUBLIC_KEY_FILE are the required bits

python3 snatch.py --dry-run
python3 snatch.py
```

## Running it for real

This is meant to run unattended for days, so run it on any always-on box you already have (a $0 free instance somewhere else, a home server, a Raspberry Pi). An example systemd unit is included:

```bash
sudo cp oracle-snatch.service /etc/systemd/system/
sudo systemctl enable --now oracle-snatch
tail -f snatch.log
```

## Things I learned the hard way

- The free A1 allowance is a total, not per-instance: 4 OCPU and 24 GB across everything. One 4/24 box, or two 2/12 boxes, after that you get LimitExceeded and the script stops on purpose.
- Upgrading the account to PAYG does not remove the free allowance. Oracle still will not charge for A1 usage inside the free total, and the capacity situation feels slightly less hopeless. My account is PAYG and the bill is still $0, with a budget alert set just in case.
- Boot volume counts toward the 200 GB free block storage total. 50 GB is plenty for a small server, do not default to 200 "because it is free", you will want the rest for a second box.
- Capacity tends to appear at odd hours. That is the whole point of a loop: it is awake at 4 a.m. so you do not have to be.
- Do not shrink the retry interval to seconds. You will not get the machine faster, you will get 429s, and then you get nothing at all for a while.

## Files

- `snatch.py` - the whole tool, one file
- `config.example.env` - all settings, commented
- `oracle-snatch.service` - example systemd unit

## License

MIT
