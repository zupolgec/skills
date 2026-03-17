---
name: qontoctl
description: >-
  Manage Qonto business banking via the qontoctl CLI. Use when the user wants to
  list accounts and balances, search transactions, manage SEPA transfers and
  beneficiaries, handle client invoices and quotes, view bank statements, manage
  team memberships, or interact with any Qonto banking feature from the command line.
license: MIT
metadata:
  author: zupolgec
  version: "0.0.1"
---

# QontoCtl

The `qontoctl` CLI lets you manage [Qonto](https://qonto.com) business banking from the terminal. It covers the full Qonto API: accounts, transactions, transfers, invoices, quotes, beneficiaries, memberships, and more.

Docs: https://qontoctl.org/

## Prerequisites

Check that the CLI is installed:

```bash
qontoctl --version
```

If not installed:

```bash
npm install -g qontoctl
```

Or use via npx (all commands below work with `npx qontoctl` too):

```bash
npx qontoctl --help
```

## Authentication

**IMPORTANT:** Before running any command, first check for existing profiles:

```bash
qontoctl profile list
```

If profiles exist, use `--profile <name>` with every command. If the user hasn't specified which profile to use and there are multiple, ask them which one to use. If there is only one profile, use it automatically.

If no profiles exist and a command fails with an auth error, guide the user through creating one:

```bash
qontoctl profile add <name>
```

QontoCtl supports two auth methods:

- **API Key** — read-only access
- **OAuth 2.0** — full access including write operations

### Profile setup

```bash
# Create a named profile (interactive)
qontoctl profile add mycompany

# Test the connection
qontoctl profile test --profile mycompany

# List all profiles
qontoctl profile list

# Show profile details (secrets redacted)
qontoctl profile show mycompany

# Remove a profile
qontoctl profile remove mycompany
```

### OAuth setup

```bash
qontoctl auth setup        # Configure OAuth client credentials
qontoctl auth login        # Start OAuth login flow
qontoctl auth status       # Display token status
qontoctl auth refresh      # Refresh access token
qontoctl auth revoke       # Revoke consent and clear tokens
```

### Config resolution order

Without `--profile`: env vars > `.qontoctl.yaml` in cwd > `~/.qontoctl.yaml`
With `--profile acme`: env vars `QONTOCTL_ACME_*` > `~/.qontoctl/acme.yaml`

## Global options

All commands accept these options:

| Option | Description |
|--------|-------------|
| `-p, --profile <name>` | Profile to use |
| `-o, --output <format>` | Output: `table` (default), `json`, `yaml`, `csv` |
| `--page <number>` | Fetch a specific page |
| `--per-page <number>` | Results per page |
| `--no-paginate` | Disable auto-pagination |
| `--verbose` | Enable verbose output |
| `--debug` | Full debug output (may log financial data) |

Always use `--output json` when you need to parse the output programmatically.

## Commands

### Organization

```bash
qontoctl org show                              # Show organization details
```

### Accounts

```bash
qontoctl account list                          # List all bank accounts
qontoctl account show <id>                     # Show account details
qontoctl account create                        # Create a new bank account
qontoctl account update <id>                   # Update an account
qontoctl account close <id>                    # Close an account
qontoctl account iban-certificate <id>         # Download IBAN certificate PDF
```

### Transactions

```bash
qontoctl transaction list                      # List transactions (supports filters)
qontoctl transaction show <id>                 # Show transaction details
qontoctl transaction attachment list <id>      # List attachments for a transaction
qontoctl transaction attachment add <id> <file>       # Attach a file
qontoctl transaction attachment remove <id> [att-id]  # Remove attachment(s)
```

### Bank statements

```bash
qontoctl statement list                        # List bank statements
qontoctl statement show <id>                   # Show statement details
qontoctl statement download <id>               # Download statement PDF
```

### Labels

```bash
qontoctl label list                            # List all labels
qontoctl label show <id>                       # Show label details
```

### Memberships

```bash
qontoctl membership list                       # List all members
qontoctl membership show                       # Show current user's membership
qontoctl membership invite                     # Invite a new member
```

### SEPA beneficiaries

```bash
qontoctl beneficiary list                      # List beneficiaries
qontoctl beneficiary show <id>                 # Show beneficiary details
qontoctl beneficiary add                       # Create a beneficiary
qontoctl beneficiary update <id>               # Update a beneficiary
qontoctl beneficiary trust <id...>             # Trust beneficiary(ies)
qontoctl beneficiary untrust <id...>           # Untrust beneficiary(ies)
```

### SEPA transfers

```bash
qontoctl transfer list                         # List transfers
qontoctl transfer show <id>                    # Show transfer details
qontoctl transfer create                       # Create a transfer
qontoctl transfer cancel <id>                  # Cancel a pending transfer
qontoctl transfer proof <id>                   # Download transfer proof PDF
qontoctl transfer verify-payee                 # Verify a payee (VoP)
qontoctl transfer bulk-verify-payee            # Bulk verify payees from CSV
```

### Internal transfers

```bash
qontoctl internal-transfer create              # Transfer between own accounts
```

### Bulk transfers

```bash
qontoctl bulk-transfer list                    # List bulk transfer batches
qontoctl bulk-transfer show <id>               # Show bulk transfer details
```

### Recurring transfers

```bash
qontoctl recurring-transfer list               # List recurring transfers
qontoctl recurring-transfer show <id>          # Show recurring transfer details
```

### Clients

```bash
qontoctl client list                           # List clients
qontoctl client show <id>                      # Show client details
qontoctl client create                         # Create a client
qontoctl client update <id>                    # Update a client
qontoctl client delete <id>                    # Delete a client
```

### Client invoices

```bash
qontoctl client-invoice list                   # List invoices
qontoctl client-invoice show <id>              # Show invoice details
qontoctl client-invoice create                 # Create a draft invoice
qontoctl client-invoice update <id>            # Update a draft invoice
qontoctl client-invoice delete <id>            # Delete a draft invoice
qontoctl client-invoice finalize <id>          # Finalize and assign number
qontoctl client-invoice send <id>              # Send invoice via email
qontoctl client-invoice mark-paid <id>         # Mark as paid
qontoctl client-invoice unmark-paid <id>       # Unmark paid status
qontoctl client-invoice cancel <id>            # Cancel a finalized invoice
qontoctl client-invoice upload <id> <file>     # Upload a file to an invoice
qontoctl client-invoice upload-show <id> <upload-id>  # Show upload details
```

### Quotes

```bash
qontoctl quote list                            # List quotes
qontoctl quote show <id>                       # Show quote details
qontoctl quote create                          # Create a quote
qontoctl quote update <id>                     # Update a quote
qontoctl quote delete <id>                     # Delete a quote
qontoctl quote send <id>                       # Send quote via email
```

### Credit notes

```bash
qontoctl credit-note list                      # List credit notes
qontoctl credit-note show <id>                 # Show credit note details
```

### Supplier invoices

```bash
qontoctl supplier-invoice list                 # List supplier invoices
qontoctl supplier-invoice show <id>            # Show supplier invoice details
qontoctl supplier-invoice bulk-create          # Create invoices from files
```

### Requests

```bash
qontoctl request list                          # List all requests
```

### Attachments

```bash
qontoctl attachment upload <file>              # Upload an attachment (PDF, JPEG, PNG)
qontoctl attachment show <id>                  # Show attachment details
```

### E-Invoicing

```bash
qontoctl einvoicing settings                   # Show e-invoicing settings
```

## Workflows

### Check account balances

```bash
qontoctl account list --output json --profile mycompany
```

### Find transactions by amount

```bash
qontoctl transaction list --output json --profile mycompany
```

Then filter the JSON output for the desired amount range.

### Create and send an invoice

1. Make sure a client exists (or create one):
   ```bash
   qontoctl client list --profile mycompany
   qontoctl client create --profile mycompany
   ```
2. Create a draft invoice:
   ```bash
   qontoctl client-invoice create --profile mycompany
   ```
3. Review it:
   ```bash
   qontoctl client-invoice show <id> --profile mycompany
   ```
4. Finalize (assigns invoice number):
   ```bash
   qontoctl client-invoice finalize <id> --profile mycompany
   ```
5. Send to client:
   ```bash
   qontoctl client-invoice send <id> --profile mycompany
   ```
6. Mark as paid when payment is received:
   ```bash
   qontoctl client-invoice mark-paid <id> --profile mycompany
   ```

### Create and send a quote

1. Ensure a client exists:
   ```bash
   qontoctl client list --profile mycompany
   ```
2. Create the quote:
   ```bash
   qontoctl quote create --profile mycompany
   ```
3. Send to client:
   ```bash
   qontoctl quote send <id> --profile mycompany
   ```

### Make a SEPA transfer

1. Add a beneficiary (if new):
   ```bash
   qontoctl beneficiary add --profile mycompany
   ```
2. Optionally trust the beneficiary:
   ```bash
   qontoctl beneficiary trust <id> --profile mycompany
   ```
3. Create the transfer:
   ```bash
   qontoctl transfer create --profile mycompany
   ```
4. Download proof when completed:
   ```bash
   qontoctl transfer proof <id> --profile mycompany
   ```

### Transfer between your own accounts

```bash
qontoctl internal-transfer create --profile mycompany
```

### Upload supplier invoices in bulk

```bash
qontoctl supplier-invoice bulk-create --profile mycompany
```

### Download a bank statement

```bash
qontoctl statement list --profile mycompany
qontoctl statement download <id> --profile mycompany
```

### Invite a team member

```bash
qontoctl membership invite --profile mycompany
```

## Tips

- Use `--output json` whenever you need to process output programmatically or when the agent needs structured data.
- Use `--profile` to switch between multiple Qonto organizations.
- Many write commands (transfers, invoices) are interactive — they will prompt for required fields.
- OAuth is required for write operations. API Key only gives read access.
- Use `--verbose` to troubleshoot API issues, but avoid `--debug` in shared environments as it may log financial data.
