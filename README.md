# 🌐 Liferay Devcon 2025 — *Make Your Data Portable*

Welcome to the official repository for the **“Make your data portable”** workshop from **Liferay Devcon 2025**.  
Inside, you'll find everything needed to follow the workshop steps or run the examples directly without changing anything in your local codebase.

---

## 📁 What’s Inside

### 🧭 Workshop Walkthrough  
A **Workshop Walkthrough** directory containing a subfolder for each step of the coding process.  Follow them in order to reproduce the full workflow demonstrated during the session.

### 📥 Test LAR File  
A **UsersAndPage.lar** file with default data you can import into the portal.  Use it to verify that your Export/Import functionality behaves as expected.

### 📦 Preconfigured Bundles  
A set of **bundles with all workshop changes already deployed**. Simply [download](https://drive.google.com/file/d/1Dboa3JAlqb3W2c2eN9UsArWObEiWVdbd
) them, update the `liferay.home` property inside `portal-ext.properties` to point to your bundle path, and run the portal directly.

---

## 🚀 Steps to Implement Export/Import for Users  

<details>
<summary><strong>Step 1 - Add new portlet for displaying Users at Export/Import</strong></summary>

To enable the display of "Users" in the Export/Import UI, we first need to register a specific portlet for this purpose.

#### 1. Create the `UserAdminPortlet` class
**File:** `modules/apps/users-admin/users-admin-web/src/main/java/com/liferay/users/admin/web/internal/portlet/UserAdminPortlet.java`

Create a new `MVCPortlet` with `category.hidden`. This ensures it doesn't appear in the widget menu but acts as the endpoint for the UI.

```java
package com.liferay.users.admin.web.internal.portlet;

import com.liferay.portal.kernel.portlet.bridges.mvc.MVCPortlet;
import com.liferay.users.admin.constants.UsersAdminPortletKeys;
import jakarta.portlet.Portlet;
import org.osgi.service.component.annotations.Component;

/**
 * @author Alberto Javier Moreno Lage
 */
@Component(
    property = {
        "com.liferay.portlet.display-category=category.hidden",
        "com.liferay.portlet.preferences-owned-by-group=true",
        "com.liferay.portlet.preferences-unique-per-layout=false",
        "com.liferay.portlet.private-request-attributes=false",
        "com.liferay.portlet.private-session-attributes=false",
        "com.liferay.portlet.use-default-template=true",
        "jakarta.portlet.display-name=Users",
        "jakarta.portlet.expiration-cache=0",
        "jakarta.portlet.name=" + UsersAdminPortletKeys.USER_ADMIN,
        "jakarta.portlet.resource-bundle=content.Language",
        "jakarta.portlet.security-role-ref=administrator",
        "jakarta.portlet.version=4.0"
    },
    service = Portlet.class
)
public class UserAdminPortlet extends MVCPortlet {
}

```

#### 2. Add the Portlet Key

**File:** `modules/apps/users-admin/users-admin-api/src/main/java/com/liferay/users/admin/constants/UsersAdminPortletKeys.java`

Register the new portlet key constant so it can be referenced by the component above.

```java
public class UsersAdminPortletKeys {
    
    // ... existing keys ...

    public static final String USER_ADMIN =
        "com_liferay_users_admin_web_portlet_UserAdminPortlet";

    // ... existing keys ...
}

```

</details>

<details>
<summary><strong>Step 2 - Enable export.import.vulcan.batch.engine.task.item.delegate property</strong></summary>

In order to register the proper `BatchEnginePortletDataHandler` for the `UsersAccountsResourceImpl`, we need to add the boolean OSGI property `export.import.vulcan.batch.engine.task.item.delegate` and set it to true.

#### Access the `UsersAccountsResourceImpl` class and change the `@Component` annotation to add the new property.
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/src/main/java/com/liferay/headless/admin/user/internal/resource/v1_0/UserAccountResourceImpl.java`

```java

//...

/**
 * @author Javier Gamarra
 */
@Component(
	properties = "OSGI-INF/liferay/rest/v1_0/user-account.properties",
	property = {
		"export.import.vulcan.batch.engine.task.item.delegate=true",
		"nested.field.support=true"
	},
	scope = ServiceScope.PROTOTYPE, service = UserAccountResource.class
)
public class UserAccountResourceImpl
//..

```
</details>

<details>
<summary><strong>Step 3 - Implement ExportImportVulcanBatchEngineTaskItemDelegate interface and its methods in UserAccountResourceImpl</strong></summary>

In order to give some more necessary context to the `BatchEnginePortletDataHandler` to handle the Exporting/Importing processes for the entity (in this case `UserAccounts`), we have a Java Interface called `ExportImportVulcanBatchEngineTaskItemDelegate` that we need to implement in each resource that will be used for that purpose.

#### 1. Access the `UsersAccountsResourceImpl` class and add the implementation of the `ExportImportVulcanBatchEngineTaskItemDelegate` interface.
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/src/main/java/com/liferay/headless/admin/user/internal/resource/v1_0/UserAccountResourceImpl.java`

```java

//...
public class UserAccountResourceImpl
	extends BaseUserAccountResourceImpl
	implements ExportImportVulcanBatchEngineTaskItemDelegate<UserAccount> {

	@Override
//..

```

#### 2.  Implement the abstract `getExportImportDescriptor()` method in `UsersAccountsResourceImpl`.
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/src/main/java/com/liferay/headless/admin/user/internal/resource/v1_0/UserAccountResourceImpl.java`

```java

//...
				_expandoColumnLocalService, _expandoTableLocalService));
	}

	@Override
	public ExportImportDescriptor getExportImportDescriptor() {
		return new ExportImportDescriptor() {

			@Override
			public String getModelClassName() {
				return User.class.getName();
			}

			@Override
			public String getPortletId() {
				return UsersAdminPortletKeys.USER_ADMIN;
			}

			@Override
			public String getResourceClassName() {
				return UserAccountResource.class.getName();
			}

			@Override
			public Scope getScope() {
				return Scope.SITE;
			}

		};
	}

	@Override
	public UserAccount getMyUserAccount() throws Exception {
//..

```
</details>

<details>
<summary><strong>Step 4 - Add new permission endpoints in rest-openapi.yaml for UserAccounts</strong></summary>

There are a numer of Export/Import functionalities that require some specific implementations at the API layer such as the "Keep permissions" option. REST Builder has some upgraded capabilities in order to simplify this permissions implementation support by doing the following.

#### 1. Add new permissions endpoints for the `UserAccount` entity  at the `rest-openapi.yaml` file following the specific desired pattern
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/rest-config.yaml`

```yaml

#...
            tags: ["Ticket"]
    "/user-accounts/{userAccountId}/permissions":
        get:
            description:
                Retrieves the user account permissions.
            operationId: getUserAccountPermissionsPage
            parameters:
                -   in: path
                    name: userAccountId
                    required: true
                    schema:
                        format: int64
                        type: integer
                -   in: query
                    name: fields
                    schema:
                        type: string
                -   in: query
                    name: restrictFields
                    schema:
                        type: string
                -   in: query
                    name: roleNames
                    schema:
                        type: string
            responses:
                204:
                    content:
                        application/json: {}
                        application/xml: {}
            tags: ["UserAccount"]
        put:
            # @review
            description:
                Overrides the userAccount permissions
            operationId: putUserAccountPermissionsPage
            parameters:
                -   in: path
                    name: userAccountId
                    required: true
                    schema:
                        format: int64
                        type: integer
            requestBody:
                content:
                    application/json: {}
                    application/xml: {}
            responses:
                204:
                    content:
                        application/json: {}
                        application/xml: {}
            tags: ["UserAccount"]
    "/user-accounts/{userAccountId}/phones":
#...

```

#### 2. Set the `generatePermissions` property to true at the `rest-config.yaml` file.
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/rest-openapi.yaml`

```yaml

apiDir: ../headless-admin-user-api/src/main/java
apiPackagePath: com.liferay.headless.admin.user
application:
    baseURI: /headless-admin-user
    className: HeadlessAdminUserApplication
    name: Liferay.Headless.Admin.User
author: Javier Gamarra
clientDir: ../headless-admin-user-client/src/main/java
compatibilityVersion: 14
generatePermissions: true
javaEEPackage: jakarta
testDir: ../headless-admin-user-test/src/testIntegration/java

```
</details>

<details>
<summary><strong>Step 5 - BuildREST</strong></summary>

By executing `BuildREST`, the new permissions logic will get autogenerated. This is one of the ways we're simplifying the process of incorporating entities to the Export/Import infrastructure as well as adding Batch support.

#### Access the `modules/apps/headless/headless-admin-user/headless-admin-user-impl` directory through your terminal and execute the `buildRest` gradle task.

```bash
gw buildREST
```
</details>

<details>
<summary><strong>Step 6 - Follow 'doX' pattern (necessary step after `generatePermissions`)</strong></summary>

Previous `GET`/`POST`/`PUT` implementations that are compatible with the nested permissions logic present in the UserAccountResourceImpl need to be migrated to the equivalent protected methods that start with "do".

#### Add "do" before each `GET`/`POST`/`PUT` that has been changed to final by REST Builder and change them from public to protected.
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/src/main/java/com/liferay/headless/admin/user/internal/resource/v1_0/UserAccountResourceImpl.java`

| Public Method                                   | Protected Method                                         |
|-------------------------------------------------|-----------------------------------------------------------|
| getAccountUserAccountsPage                      | doGetAccountUserAccountsPage                              |
| getOrganizationUserAccountsPage                 | doGetOrganizationUserAccountsPage                         |
| getSiteUserAccountsPage                         | doGetSiteUserAccountsPage                                 |
| getUserAccount                                   | doGetUserAccount                                          |
| getUserAccountByExternalReferenceCode           | doGetUserAccountByExternalReferenceCode                   |
| getUserAccountsPage                              | doGetUserAccountsPage                                     |
| postAccountUserAccount                           | doPostAccountUserAccount                                  |
| postSiteUserAccount                                  | doPostSiteUserAccount                                         |
| postUserAccount                                  | doPostUserAccount                                         |
| putUserAccount                                   | doPutUserAccount                                          |
| putUserAccountByExternalReferenceCode           | doPutUserAccountByExternalReferenceCode                   |

</details>

<details>
<summary><strong>Step 7 - Implement permission checker auxiliary methods</strong></summary>

The autogenerated logic for calculating the permissions require the implementation of some auxiliary methods that exposes necessary data for the `ResourcePermissions` layer.

#### Access `UsersAccountsResourceImpl` and implement this newly added methods.
**File:** `modules/apps/headless/headless-admin-user/headless-admin-user-impl/src/main/java/com/liferay/headless/admin/user/internal/resource/v1_0/UserAccountResourceImpl.java`

```java

//...
	}

	@Override
	protected Long getPermissionCheckerGroupId(Object id) throws Exception {
		User user = _userService.getUserById((Long)id);

		return user.getGroupId();
	}

	@Override
	protected String getPermissionCheckerPortletName(Object id) {
		return "com.liferay.portal.kernel.model.User";
	}

	@Override
	protected String getPermissionCheckerResourceName(Object id) {
		return User.class.getName();
	}

	@Override
	protected void preparePatch(
//..

```
</details>

---

Feel free to explore, tweak, and build on top of these resources—enjoy the workshop!
