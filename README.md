# Setup AWS Cognito User Pool with an Azure AD identity provider to perform single sign-on (SSO) authentication in mobile app (Part 1).

This a step-by-step tutorial of how to set up an AWS Cognito User Pool with an Azure AD identity provider and perform single sign-on (SSO) authentication with Azure AD account to access AWS services in your iOS and Android mobile application. Tutorial will consist of 3 separate parts:

1.  Setup AWS Cognito User Pool with an Azure AD identity provider to perform single sign-on (SSO) authentication with mobile app.
2.  Integration Cognito Auth in iOS application.
3.  Integration Cognito Auth in Android application.
#### Glossary
[Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html) - service that provides authentication, authorization, and user management for web and mobile apps. Users can sign-in directly with a username and password or through a third party such as Azure AD, Amazon, or Google. Amazon Cognito consists of two main components: user pools and identity pools. User pools are user directories that provide sign-up and sign-in options for app users. Identity pools enable you to grant your users access to other AWS services. You can use identity pools and user pools separately or together.

[Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-whatis) (Azure Active Directory) - Microsoft’s multi-tenant, cloud-based directory, and identity management service.

[Federation Identity Management (FIdM)](https://en.wikipedia.org/wiki/Federated_identity_management) - a system of shared protocols, technologies and standards that allows user identities and devices to be managed across organizations.

[Identity Provider (IdP)](https://en.wikipedia.org/wiki/Identity_provider) - a system that creates, maintains, and manages identity information for principals (users, services, or systems) and provides principal authentication to other service providers (applications) within a federation or distributed network. An IdP can provide a user with identifying information and serve that information to services when the user requests access.

[SAML (Security Assertion Markup Language)](https://en.wikipedia.org/wiki/SAML_2.0) - is a standard  for securely exchanging user’s identity between SAML authority (called an identity provider or IdP) and SAML consumer (called a service provider or SP). Thus defining 3 roles: the principal (user), identity provider and service provider.  SAML eliminates passing passwords. Instead, it uses cryptography and digital signatures to pass a secure sign-in token from an identity provider to a service provider. Is one of the most widely used protocols when it comes to Single sign-on implementation.

[Service Providers (SP)](https://en.wikipedia.org/wiki/Service_provider_(SAML)) - an entity that provides Web Services that receives and accepts authentication assertions in conjunction with a [single sign-on](https://en.wikipedia.org/wiki/Single_sign-on) (SSO) profile of the [Security Assertion Markup Language](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language) (SAML). SAML’s Service Provider (SP) depends on receiving assertions from a SAML Identity Provider (IdP).

[Single sign-on (SSO)](https://en.wikipedia.org/wiki/Single_sign-on) - is an authentication process which allows automatically granting access to multiple system services and apps by once log in to the system. Single sign-on typically use in enterprise environments by providing employees single access to the services and applications rather than creating and managing separate credentials for each service.
#
In case SSO authentication with Azure AD account to AWS Cognito, Azure AD will be an identity provider (IdP) and AWS Cognito a Service provider (SP). AWS Cognito before giving to the user an access to AWS resources checks with the identity provider if the user's permissions. Azure AD verifies user identity (emails and password, for example) and if valid asserts back to AWS Cognito that user should have access along with the user's identity.


![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image10.png)

 
The SSO flow based on the next steps:

1.  The user accesses an application, which redirects him to a page hosted by AWS Cognito.
    
2.  AWS Cognito identifies the user’s origin (by client id, application subdomain etc) and redirects the user to the identity provider, asking for authentication. In this case to an Azure AD login page. This is the SAML authentication request.
    
3.  The browser redirects the user to an SSO URL. The user either has an existing active browser session with the identity provider or establishes one by logging into the identity provider. The identity provider (Azure AD) creates the authentication response in the XML-document format, which contains the user’s username or email address (and other attributes if set) and signs it using an X.509 certificate. The result is passing back to the service provider (AWS Cognito). This is the SAML authentication response.
    
4.  The service provider, which already knows the identity provider and has a certificate fingerprint, retrieves the authentication response and validates it using the certificate fingerprint. The identity of the user is established and the user is provided with app access. with the access_token in the URL.

#### Requirements list:

1.  Azure account with Azure AD Premium enabled.
2.  AWS account.

#### The setup consist of 5 steps:

1.  Create an AWS Cognito User Pool.
2.  Create AWS App client and add it to the User Pool.
3.  Create an Azure AD enterprise application.
4.  Set up Azure AD identity provider to the Cognito User Pool.
5.  Setup Identity Provider in your AWS User Pool.
    
## 1.  Create a AWS Cognito User Pool
    
1.1 Login to AWS Console ([https://console.aws.amazon.com/](https://console.aws.amazon.com/)) and open “All Services” section.

1.2 Choose “Cognito” in section Security, Identity & Compliance:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image28.png)

1.3 In Cognito service choose “Manage User Pools”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image1.png)

1.4 Choose “Create a user pool”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image4.png)

1.5 Type a name of your user pool and choose “Review Defaults” in case you don’t have specific settings you want to set:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image12.png)

1.6 Choose section with required attributes and click on edit:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image22.png)

1.7 Setup user sign-in option by choosing “email address or phone number”. In subcategories choose “allow email addresses” and choose “Next step”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image27.png)

1.8 Leave all settings default (if you don’t want to set some). At the last screen choose “Create Pool”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image15.png)

1.9 Now your pool is created. Memorize Pool Id (e.g. us-east-1_XX123xxXXX). You will need this id in Azure AD portal and mobile app settings.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image6.png)

1.10 Set User Pool Domain Name. For this open your User Pool, choose section “App Integration” -> “Domain Name”. Type your domain prefix.

Amazon Cognito Domain is built by this scheme:
```
https://<Domain prefix>.auth.<Region>.amazoncognito.com
```

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image21.png)

Memorize it, it will be required in Azure and mobile app settings.

With this example Amazon Cognito Domain is [https://example-setup-app.auth.us-east-1.amazoncognito.com](https://example-setup-app.auth.us-east-1.amazoncognito.com)

**As a result of this section you should have next information:**
-   User Pool id (e.g. us-east-1_XX123xxXXX)
-   Amazon Cognito Domain associated with User Pool (e.g. [https://example-setup-app.auth.us-east-1.amazoncognito.com](https://example-setup-app.auth.us-east-1.amazoncognito.com))
    
## 2. Create App client and add it to the User Pool

Basically, you can create your application with Mobile Hub and associate it with your user pool. But in this tutorial described how to create an application from Cognito Service.

2.1 Open your User Pool, choose “General settings” -> “App Clients” and click on “Add new app client”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image11.png)

2.2 Type a name of your app client, e.g. “iOS App Client”, make sure that “Generate client secret” is checked, leave other setting default. Press “Create app client”.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image19.png)

2.3 Now your app client is created, open “General” -> “App Clients”. Your application will be listed there. Memorize “App client id” and “App client secret”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image25.png)

2.4 Setup App Client. Open “App integration” -> “App Client Settings”. Choose your mobile client app and set next settings:

[Allowed OAuth Flows](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html): Authorization code grant, Implicit grant;

[Allowed OAuth Scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html):  email, aws.cognito.signin.user.admin, openid (openid is required with email scope);

Callback URL(s) and Sign Out URL(s) should be set to your app URL Scheme (you can read more about this here):
-   iOS: [Defining a Custom URL Scheme for Your App](https://developer.apple.com/documentation/uikit/core_app/allowing_apps_and_websites_to_link_to_your_content/communicating_with_other_apps_using_custom_urls);
-   Android: [Create Deep Links to App Content](https://developer.android.com/training/app-links/deep-linking).

Save your changes.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image23.png)

**At the end of this section you should have the next information:**
- App client id;
- App client secret;
- Callback URL;
- Sign Out URL;
- List of Allowed OAuth Scopes.

This is not all set-up which you need to perform in AWS, but for now, you need to continue with setup Azure.

## 3. Create an Azure AD enterprise application

3.1 Open Azure Portal [https://portal.azure.com/](https://portal.azure.com/), on the right side menu choose “Azure Active Directory”.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image18.png)

If there is no such service, Open “All services” and type “Azure Active Directory”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image5.png)

3.2 In Active Directory menu choose “Enterprise applications”:  

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image20.png)

3.3 In opened section choose “New Application”:  

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image30.png)

  

3.4 Pick “Non-gallery application” type for your application:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image16.png)

  

3.5 Type name of your application and press “Add”. Now your application is created and time to connect it to AWS User Pool.

  

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image31.png)

3.6 Setup Single sign-on. In your Azure AD enterprise application choose section “Single sign-on”, in dropdown list choose “SAML-based Sign-on”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image9.png)

In section “Domain and URLs” set next information:

-   **Identifier**. Identifier contains your User Pool id (from AWS) and built with next pattern:
```
urn:amazon:cognito:sp:<yourUserPoolID>  
```

-   **Reply URL**. The Reply URL is where from application expects to receive the authentication token. This is also referred to as the “Assertion Consumer Service” (ACS) in SAML. Is should follow the pattern:
```
https://<yourDomainPrefix>.auth.<yourRegion>.amazoncognito.com/saml2/idpresponse
```

Example of Identifier and Reply URL:

Identifier: urn:amazon:cognito:sp:us-east-1_XX123xxXXX

Reply URL: ```https://example-setup-app.auth.us-east-1.amazoncognito.com/saml2/idpresponse```

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image8.png)

Save your changes and download SAML File:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image17.png)

3.7 Add a User to your app. In your Azure AD select “Enterprise applications” and choose your application. Select “Users and groups”->“Add user”.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image14.png)

Invite new users or select from existing. These users will be able to login with this Azure AD account to your application. When you’ll finish adding a user select “Assign”.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image24.png)

**This is all settings in the Azure portal. At the end of this section you should have:**
-   SAML file with XML format;
-   user(s) to login.

## 5. Setup Identity Provider in your AWS User Pool

5.1 Open your User Pool and choose section “Federation” -> “Identity Providers”. In opened section select “SAML” provider:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image3.png)

5.2 Type a name for your provider and upload SAML file from Azure. Press “Create Provider”:

  

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image32.png)

5.3 Setup attribute mapping from your provider to AWS. In this example we are only interested in email, so for email add next:

**SAML Attribute**:
```http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress```

**User pool attribute**: Email

Save your changes.


![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image26.png)


5.4 Assign Identity provider to your app client. In your user pool open section “App Client Settings”. Choose your application, in the section “Enabled Identity Providers” choose a provider which you just created for this user pool. Save your changes.

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image13.png)

That’s all settings which you should do in AWS console and Azure portal. You can now test your set-up.

## Testing your set-up

You can easily test your setup in Azure Portal:
1.  Open Single sign-on section of your application in the Azure portal and choose button “Test SAML Settings”:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image7.png)

2. Then you will need to install My Apps Secure Sign-in Extension and the perform a sign in with the account which you have added to this application on step 3.7:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image2.png)

3. You will be able to see SAML request and response, and token if the login succeeds:

![](https://github.com/2ZGroupSolutionsArticles/Article_KZ001/blob/master/Images/image29.png)

At this point, you should have all required values to begin setup SSO authentication with Azure AD account in your mobile application.

The final list of settings which you should have at the end of this setup:
-   User Pool ID;
-   Amazon Cognito Domain associated with User Pool;
-   App client ID;
-   App client secret;
-   Callback URL;
-   Sign Out URL;
-   List of Allowed OAuth Scopes;
-   Region.
    
**Additional resources:**

1.  [https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html)    
2.  https://docs.aws.amazon.com/singlesignon/latest/userguide/samlfederationconcept.html
3.  [https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-saml-idp.html](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-saml-idp.html)
4.  [https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/configure-single-sign-on-non-gallery-applications#configuring-and-testing-azure-ad-single-sign-on](https://docs.microsoft.com/en-us/azure/active-directory/manage-apps/configure-single-sign-on-non-gallery-applications#configuring-and-testing-azure-ad-single-sign-on)
5.  [https://docs.microsoft.com/en-us/azure/active-directory/saas-apps/tutorial-list](https://docs.microsoft.com/en-us/azure/active-directory/saas-apps/tutorial-list)
6.  [https://aws.amazon.com/blogs/mobile/amazon-cognito-user-pools-supports-federation-with-saml](https://aws.amazon.com/blogs/mobile/amazon-cognito-user-pools-supports-federation-with-saml)
7.  [https://docs.microsoft.com/en-us/azure/active-directory/active-directory-enterprise-apps-manage-sso](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-enterprise-apps-manage-sso)
8.  [https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-token-and-claims](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-token-and-claims)
9.  [https://go.microsoft.com/fwLink/?LinkID=717349#configuring-and-testing-azure-ad-single-sign-on](https://go.microsoft.com/fwLink/?LinkID=717349#configuring-and-testing-azure-ad-single-sign-on)
10.  [https://docs.microsoft.com/en-us/azure/active-directory/active-directory-enterprise-apps-manage-sso](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-enterprise-apps-manage-sso)  

Author:

Kseniia Zozulia

Email: kseniiazozulia@2zgroup.net

LinkedIn: [Kseniia Zozulia](https://www.linkedin.com/in/629bb187)
