# Your own server in Azure

For the next lessons you need a Linux server on the internet. Azure for Students
gives you one for free: $100 of credit for 12 months, no credit card, only your
college email. This takes about 20 minutes.

By the end you can type `ssh azureuser@<your-ip>` on your laptop and you are on
your own Ubuntu server, as its administrator.

## 1 · Get Azure for Students

1. Open <https://azure.microsoft.com/free/students> and click **Start free**.
2. Sign in with your **college email**. If Microsoft does not know this address yet,
   it asks you to create a Microsoft account with it; do that.
3. Confirm that you are a student and finish the form. You do **not** need a card.

Checkpoint: <https://portal.azure.com> shows a subscription called **Azure for Students**.

> [!NOTE]
> The credit is real money that Microsoft pays for you. When it is gone, the server
> stops. One small server uses only a small part of it, so do not create servers
> you do not need.

## 2 · Open Cloud Shell

Open <https://shell.azure.com>, choose **Bash**. If it asks about storage, choose
**No storage account required** and your **Azure for Students** subscription.

Cloud Shell is a Linux terminal in the browser that is already logged in to your
Azure account. Everything below runs there, **not** on your laptop, until step 6.

## 3 · Find your regions

Your subscription may create servers in only five regions. Find them:

```bash
az policy assignment list --query "[].parameters.listOfAllowedLocations.value" --output tsv
```

If the list is empty, open the portal, search for **Policy**, then
**Authoring → Assignments → Allowed resource deployment regions**, and read the
regions there.

Pick one region, for example the first one, and check that the free server size is
allowed there:

```bash
REGION=<a region from the list>
az vm list-skus --location $REGION --size Standard_B2ats_v2 --output table
```

Checkpoint: one line with `Standard_B2ats_v2`, and the column **Restrictions** says
`None`. If it says `NotAvailableForSubscription`, try the next region.

## 4 · Create the server

You need the **public** half of the SSH key from Lesson 2. On your laptop (WSL):

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the whole line. Back in Cloud Shell, put it between the quotes:

```bash
MY_KEY="ssh-ed25519 AAAA... wsl-laptop"
```

Now create a resource group (a folder for everything that belongs to this server)
and the server itself:

```bash
az group create --name devsecops --location $REGION

az vm create \
  --resource-group devsecops \
  --name conduit-server \
  --image Ubuntu2404 \
  --size Standard_B2ats_v2 \
  --admin-username azureuser \
  --ssh-key-values "$MY_KEY" \
  --public-ip-sku Standard
```

It takes one or two minutes. Checkpoint: the output has `"powerState": "VM running"`
and a `publicIpAddress`. Write the address down.

> [!NOTE]
> You did not click through forms: you described the server in one command. Next
> time you need the same server, you run the same command. That is the first step of
> infrastructure as code.

## 5 · Open the port for Conduit

Only SSH (port 22) is open now. Open port 8000 for the API:

```bash
az vm open-port --resource-group devsecops --name conduit-server --port 8000 --priority 1010
```

## 6 · Log in from your laptop

On your laptop (WSL):

```bash
ssh azureuser@<your-ip>
```

Type `yes` the first time. Then, on the server:

```bash
hostnamectl
sudo whoami
```

Checkpoint: Ubuntu 24.04, and `sudo whoami` answers `root`. This whole machine is yours.

> [!WARNING]
> Your server is on the internet. Bots start trying to log in within minutes. It
> accepts only your SSH key, never a password: keep it that way, and never share
> the private key (the file without `.pub`).

## When you do not need the server any more

At the end of the course delete everything with one command in Cloud Shell:

```bash
az group delete --name devsecops
```

## Troubleshooting

1. **The sign-up does not accept the college email.** Try once more in a private
   browser window. If it still fails, tell the instructor; do not use a card.
2. **`RequestDisallowedByAzure`.** The region is not one of your five. Repeat step 3.
3. **`SkuNotAvailable` or `NotAvailableForSubscription`.** This size is not available
   in this region. Try another of your regions, or the size `Standard_B2ts_v2`.
4. **`QuotaExceeded`.** Your subscription allows only a few processor cores. Delete
   servers you do not need: `az vm list --output table`.
5. **`Invalid image "Ubuntu2404"`.** Use the full name instead:
   `--image Canonical:ubuntu-24_04-lts:server:latest`.
6. **`ssh` from the laptop hangs.** Check the address. If it still hangs, the network
   may block port 22; tell the instructor.
7. **`Permission denied (publickey)`.** The key in step 4 is not the one on your
   laptop. Compare the end of both lines.
