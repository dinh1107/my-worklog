---
title: "Creating an IAM User and Configuring Permissions"
date: 2026-09-14
weight: 2
chapter: false
pre: "<b>5.2. </b>"
---

# Creating an IAM User and Configuring Permissions

In this section, we will create the `SmartHome-Developers` IAM group, the `dinh-fcj` IAM user, AWS Management Console access, and MFA protection.

The IAM user will replace the root user for the remaining deployment steps.

## Why is IAM required?

The root user has complete access to the AWS account and should not be used for daily operations.

AWS Identity and Access Management makes it possible to:

- Create a separate identity for the project administrator.
- Manage access through policies.
- Group users with similar responsibilities.
- Apply the principle of least privilege.
- Modify or revoke permissions without affecting the root user.

In this project, permissions are managed through an IAM group instead of attaching every policy directly to an individual user.

## Objectives

After completing this section:

- The `SmartHome-Developers` IAM group is available.
- The `dinh-fcj` IAM user is available.
- The user belongs to the correct group.
- AWS Management Console access is enabled.
- No unnecessary access key is created.
- MFA is enabled for the IAM user.
- The user can sign in and work through `dinh-fcj`.

---

## 5.2.1. Open IAM

Sign in to the AWS Management Console as the root user or another identity with IAM administration permissions.

Enter the following in the search bar:

```text
IAM
```

Select:

```text
Identity and Access Management (IAM)
```

IAM is a global service and does not depend on the currently selected Region.

---

## 5.2.2. Create the IAM group

From the IAM navigation pane, select:

```text
User groups
→ Create group
```

Enter the group name:

```text
SmartHome-Developers
```

This group centrally manages permissions for users responsible for deploying the project.

Policies do not have to be attached at this point. Service-specific policies will be added during the relevant AWS deployment sections.

Select:

```text
Create group
```

### Verify the result

The group should appear in the **User groups** list.

![SmartHome-Developers IAM group created](/images/5.2.2.png)

### Why use an IAM group?

When a policy is attached to a group, all members receive the corresponding permissions. This approach provides:

- Centralized permission management.
- No repeated policy assignments for multiple users.
- Easier permission updates and revocation.
- Consistent access for users with the same responsibilities.

---

## 5.2.3. Create the IAM user

From the IAM navigation pane, select:

```text
Users
→ Create user
```

Enter the user name:

```text
dinh-fcj
```

Select:

```text
Provide user access to the AWS Management Console
```

If AWS asks for the user type, select:

```text
I want to create an IAM user
```

Configure an initial password. It can be automatically generated or manually specified according to the account password policy.

Enable:

```text
Users must create a new password at next sign-in
```

This requires the user to replace the temporary password during the first sign-in.



Select **Next** to continue to the permission configuration.

---

## 5.2.4. Add the user to the group

Under **Set permissions**, select:

```text
Add user to group
```

Select:

```text
SmartHome-Developers
```

Select **Next**, review the information, and select:

```text
Create user
```

### Verify the result

Open:

```text
IAM
→ Users
→ dinh-fcj
```

Under **Groups**, the user should be a member of:

```text
SmartHome-Developers
```

![dinh-fcj belongs to SmartHome-Developers](/images/5.2.4.png)

---

## 5.2.5. Configure group permissions

Open:

```text
IAM
→ User groups
→ SmartHome-Developers
→ Permissions
```

The permissions of `dinh-fcj` are primarily managed through customer managed policies attached to this group.

Policies are added progressively during the workshop:

| Permission group | Purpose |
|---|---|
| AWS IoT Core | Manages Things, certificates, policies, and IoT Rules |
| AWS Lambda | Creates and configures Lambda functions |
| DynamoDB | Creates the table and inspects stored data |
| Amazon SNS | Creates the topic and email subscription |
| CloudWatch | Reads logs and Lambda metrics |
| Amazon S3 | Creates the bucket and inspects reports |
| EventBridge Scheduler | Creates the daily Lambda schedule |
| IAM PassRole | Passes an approved execution role to an AWS service |

![Policies attached to SmartHome-Developers](/images/5.2.5.png)

### Permission principles

The policies should follow the principle of least privilege:

- Grant only the actions required by the project.
- Restrict permissions to specific resources where possible.
- Do not use `AdministratorAccess` for daily operations.
- Do not grant `iam:PassRole` for every IAM role.
- Do not allow management of root access keys.



---

## 5.2.6. Check the user credentials

Open:

```text
IAM
→ Users
→ dinh-fcj
→ Security credentials
```

Verify:

```text
Console access: Enabled
```

This workshop primarily uses the AWS Management Console, so an access key is not required for `dinh-fcj`.

Recommended result:

```text
Access keys: None
```

An access key should only be created when AWS CLI, SDK, or API access is required.

![IAM user security credentials](/images/5.2.6.png)


---

## 5.2.7. Enable MFA for the IAM user

Under the **Security credentials** tab, locate:

```text
Multi-factor authentication (MFA)
```

Select:

```text
Assign MFA device
```

Enter a device name, for example:

```text
dinh-fcj-MFA
```

Select:

```text
Authenticator app
```

Then:

1. Open the authenticator application.
2. Scan the QR code provided by AWS.
3. Enter two consecutive MFA codes.
4. Select **Add MFA**.



---

## 5.2.8. Sign in as the IAM user

Sign out of the root user and open the IAM sign-in URL for the account.

The URL usually has the following format:

```text
https://<ACCOUNT-ALIAS-OR-ID>.signin.aws.amazon.com/console
```

Enter:

```text
IAM user name: dinh-fcj
Password: IAM user password
MFA code: code from the authenticator application
```

If requested, change the temporary password during the first sign-in.

After signing in:

1. Verify that the account menu displays `dinh-fcj`.
2. Select `Asia Pacific (Singapore)`.
3. Do not use the root user for the remaining deployment steps.


---

## Completion checklist

- [ ] The `SmartHome-Developers` group has been created.
- [ ] The `dinh-fcj` IAM user has been created.
- [ ] The user belongs to `SmartHome-Developers`.
- [ ] AWS Console access is enabled.
- [ ] The user has no unnecessary access keys.
- [ ] Project policies are managed through the group.
- [ ] MFA is enabled for the IAM user.
- [ ] Signing in as `dinh-fcj` is successful.
- [ ] The selected Region is `ap-southeast-1`.
- [ ] The root user is no longer used for daily tasks.

## Conclusion

In this section, we created the `SmartHome-Developers` IAM group, the `dinh-fcj` IAM user, and MFA-protected console access.

This IAM user will be used to deploy and manage AWS resources in the following sections. Service-specific permissions will be added when required to maintain least-privilege access.

Next, we will define the Region, resource names, and MQTT topics used throughout the system.

## References

- [AWS – Create an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
- [AWS – Create IAM groups](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_create.html)
- [AWS – Enable MFA for an IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html)
- [AWS – IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)