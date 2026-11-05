Closes #3

## Overview

Adds the Payment Log list export and
docs/rebuild/payment-log-registry.md.

The list stores payment details, with Unclaimed as the initial
status. Payment references are generated from the saved list ID
using the PAY-00001 format.

The rebuild document describes the fields, reference formula,
status transitions, and proposed reminder rule. It also explains
that Coded does not confirm a successful NXT transfer.

## Testing

Manually tested in Établi with fictional payments:
- Saving without Gift Amount was refused.
- Saving with all required fields succeeded.
- The initial status was Unclaimed.
- The test button generated a payment reference from the saved ID.

## Screenshots / Screencasts

None attached.

## Checklist

- [ ] Issue number is linked
- [ ] Branch is linked
- [ ] Reviewers are assigned (one of your tech leads)
