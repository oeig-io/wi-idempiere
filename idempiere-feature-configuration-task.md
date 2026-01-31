# Feature Configuration Task

The purpose of this task is to create and maintain iDempiere configuration artifacts in test environments.

## Access Points

- **Logs**: /opt/idempiere-server/log/
- **Database**: psql
- **API**: iDempiere REST API

## Configuration Artifacts

- Window, tab, and field
- Table and column
- Menu
- Role
- Groovy (callouts, processes, events)

## Common Workflow

1. Create groovy artifact
1. Test artifact by observing logs and db records
1. Iterate until behavior is correct
1. Migrate to pristine (see idempiere-environment-management-task)

Tags: #role-system-admin
