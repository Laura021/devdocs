---
group: web-api
title: Create an integration
redirect_from: /guides/v2.1/howdoi/webapi/integration.html
---


An **integration** enables third-party services to call the Magento web APIs. The Magento APIs currently supports Accounting, Enterprise Resource Planning (ERP), Customer Relationship Management (CRM), Product Information Management (PIM), and marketing automation systems out of the box.

Implementing a simple integration requires little knowledge of {% glossarytooltip bf703ab1-ca4b-48f9-b2b7-16a81fd46e02 %}PHP{% endglossarytooltip %} or Magento internal processes. However, you will need a working knowledge of

* [Magento REST or SOAP Web APIs]({{ page.baseurl }}/get-started/bk-get-started-api.html)
* [Web API authentication]({{ page.baseurl }}/get-started/authentication/gs-authentication.html)
* [OAuth-based authentication]( {{ page.baseurl }}/get-started/authentication/gs-authentication-oauth.html )


Before you begin creating a module, make sure that you have a working installation of Magento 2.0, and the [Magento System Requirements]({{ page.baseurl }}/install-gde/system-requirements.html).

To create an integration, follow these general steps:

1. [Create a module with the minimal structure and configuration.](#skeletal)
2. [Add files specific to the integration.](#files)
3. [Install the module.](#install)
4. [Check the integration.](#check)
5. [Integrate with your application.](#integrate)

## Create a skeletal module {#skeletal}

To develop a module, you must:

1. **Create the module file structure.** The module for an integration can be placed anywhere under the Magento root directory, but the recommended location is `<magento_base_dir>/vendor/<vendor_name>/module-<module_name>`.

   Also create  `etc`, `etc/integration`, and `Setup` subdirectories under `module-<module_name>`, as shown in the following example:

    ```
    cd <magento_base_dir>
    mkdir -p vendor/<vendor_name>/module-<module_name>/etc/integration
    mkdir -p vendor/<vendor_name>/module-<module_name>/Setup
   ```
   For more detailed information, see [Create your component file structure]({{ page.baseurl }}/extension-dev-guide/build/module-file-structure.html).

2. **Define your module configuration file.** The `etc/module.xml` file provides basic information about the module. Change directories to the `etc` directory and create the `module.xml` file. You must specify values for the following attributes:

    Attribute | Description
    --- | ---
    `name` | A string that uniquely identifies the module
    `setup_version` | The version of Magento the component uses
    {:style="table-layout:auto;"}

    The following example shows an example `etc/module.xml` file.

    ```xml
    <?xml version="1.0"?>
   <!--
   /**
   * Copyright © 2015 Magento. All rights reserved.
   * See COPYING.txt for license details.
   */
   -->
   <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
       <module name="Vendor1_Module1" setup_version="2.0.0">
            <sequence>
                <module name="Magento_Integration"/>
            </sequence>
       </module>
     </config>
    ```

    Module `Magento_Integration` is added to “sequence” to be loaded first. It helps to avoid the issue, when a module with integration config loaded, that leads to a malfunction.


3. **Add your module's `composer.json` file.** Composer is a dependency manager for PHP. You must create a `composer.json` file for your module so that Composer can install and update the libraries your module relies on. Place the `composer.json` file in the `module-<module_name>` directory.

    The following example demonstrates a minimal `composer.json` file.

    ``` json
      {
         "name": "Vendor1_Module1",
         "description": "create integration from config",
         "require": {
            "php": "~7.0.0",
            "magento/framework": "2.1.0",
            "magento/module-integration": "2.1.0"
         },
         "type": "magento2-module",
         "version": "1.0",
         "autoload": {
            "files": [ "registration.php" ],
            "psr-4": {
               "Vendor1\\Module1\\": ""
            }
         }
      }
    ```

    For more information, see [Create a component]({{ page.baseurl }}/extension-dev-guide/build/create_component.html).

4. **Create a `registration.php` file** The `registration.php` registers the module with the Magento system. It must be placed in the module's root directory.

      ```php
      <?php
        /**
        * Copyright © 2015 Magento. All rights reserved.
        * See COPYING.txt for license details.
        */

        \Magento\Framework\Component\ComponentRegistrar::register(
        \Magento\Framework\Component\ComponentRegistrar::MODULE,
        'Vendor1_Module1',
        __DIR__
        );
      ```

5. **Create an install class.**
Change directories to your `Setup` directory. Create a `InstallData.php` file that installs the integration configuration data into the Magento integration table.

    The following sample is boilerplate and requires minor changes to make your integration work.

    ```php
    <?php
    namespace Vendor1\Module1\Setup;

    use Magento\Framework\Setup\ModuleContextInterface;
    use Magento\Framework\Setup\ModuleDataSetupInterface;
    use Magento\Integration\Model\ConfigBasedIntegrationManager;
    use Magento\Framework\Setup\InstallDataInterface;

    class InstallData implements InstallDataInterface
    {
        /**
         * @var ConfigBasedIntegrationManager
         */


        private $integrationManager;

        /**
         * @param ConfigBasedIntegrationManager $integrationManager
         */

        public function __construct(ConfigBasedIntegrationManager $integrationManager)
        {
            $this->integrationManager = $integrationManager;
        }

        /**
         * {@inheritdoc}
         */

        public function install(ModuleDataSetupInterface $setup, ModuleContextInterface $context)
        {
            $this->integrationManager->processIntegrationConfig(['testIntegration']);
        }
    }
    ```  

    In the following line

    `$this->integrationManager->processIntegrationConfig(['testIntegration']);`

    `testIntegration` must refer to your `etc/integrations/config.xml` file, and the integration name value must be the same.

    Also, be sure to change the path after `namespace` for your vendor and module names.

## Create integration files {#files}

Magento provides the Integration module, which simplifies the process of defining your integration. This module automatically performs functions such as:

* Managing the third-party account that connects to Magento.
* Maintaining OAuth authorizations and user data.
* Managing security tokens and requests.

To customize your module, you must create multiple {% glossarytooltip 8c0645c5-aa6b-4a52-8266-5659a8b9d079 %}XML{% endglossarytooltip %} files and read through others files to determine what resources existing Magento modules have access to.

The process for customizing your module includes

* [Define the required resources](#resources)
* [Pre-configure the integration](#preconfig)

### Define the required resources {#resources}

The `etc/integration/api.xml` file defines which {% glossarytooltip 786086f2-622b-4007-97fe-2c19e5283035 %}API{% endglossarytooltip %} resources the integration has access to.

To determine which resources an integration needs access to, review the permissions defined in each module's `etc/acl.xml` file.

In the following example, the test integration requires access to the following resources in the Sales module:

```xml
<integrations>
    <integration name="testIntegration">
        <resources>
            <!-- To grant permission to Magento_Log::online, its parent Magento_Customer::customer needs to be declared as well-->
            <resource name="Magento_Customer::customer" />
            <resource name="Magento_Log::online" />
            <!-- To grant permission to Magento_Sales::reorder, all its parent resources need to be declared-->
            <resource name="Magento_Sales::sales" />
            <resource name="Magento_Sales::sales_operation" />
            <resource name="Magento_Sales::sales_order" />
            <resource name="Magento_Sales::actions" />
            <resource name="Magento_Sales::reorder" />
        </resources>
    </integration>
</integrations>
```

### Pre-configure the integration {#preconfig}

Your module can optionally provide a configuration file `config.xml` so that the integration can be automatically pre-configured with default values. To enable this feature, create the `config.xml` file in the `etc/integration` directory.

{:.bs-callout .bs-callout-tip}
If you pre-configure the integration, the values cannot be edited from the {% glossarytooltip 29ddb393-ca22-4df9-a8d4-0024d75739b1 %}admin{% endglossarytooltip %} panel.

The file defines which API resources the integration has access to.

``` xml
<integrations>
   <integration name="TestIntegration">
       <email></email>
       <endpoint_url></endpoint_url>
       <identity_link_url></identity_link_url>
   </integration>
</integrations>
```

Element | Description
--- | ---
`integrations` | Contains one or more integration definitions.
`integration name` | Defines an integration. The `name` must be specified.
`email` | An email to associate with this integration.
`endpoint_url` | Optional. The URL where OAuth credentials can be sent when using OAuth for token exchange. We strongly recommend using `https://`. See [OAuth-based authentication]({{ page.baseurl }}/get-started/authentication/gs-authentication-oauth.html) for details.
`identity_link_url` | Optional. The URL that redirects the user to link their 3rd party account with the Magento integration.
{:style="table-layout:auto;"}

## Install your module {#install}

Use the following steps to install your module:

1. Run the following command to update the Magento {% glossarytooltip 66b924b4-8097-4aea-93d9-05a81e6cc00c %}database schema{% endglossarytooltip %} and data.

    <code>bin/magento setup:upgrade</code>

2. Run the following command to generate the new code.

   {: .bs-callout .bs-callout-info }
   In Production mode, you may receive a message to 'Please rerun Magento compile command'.  Enter the command below. Magento does not prompt you to run the compile command in Developer mode.

   <code>bin/magento setup:di:compile</code>

3. Run the following command to clean the cache.

   <code>bin/magento cache:clean</code>

## Check your integration {#check}

Log in to Magento and navigate to **System > Extensions > Integrations**. The integration should be displayed in the grid.

## Integrate with your application {#integrate}

Before you can activate your integration in Magento, you must create two pages on your application to handle OAuth communications.

* The location specified in the `identity_link_url` parameter must point to a page that can handle login requests.

* The location specified in the `endpoint_url` parameter (**Callback URL** in Admin) must be able to process OAuth token exchanges.

### Login page {#login}

When a merchant clicks the **Activate** button in Admin, a pop-up login page for the third-party application displays. Magento sends values for `oauth_consumer_key` and `success_call_back` parameters. The application must store the value for`oauth_consumer_key` tie it to the login ID. Use the `success_call_back` parameter to return control back to Magento.

### Callback page {#callback}

The callback page must be able to perform the following tasks:

* Receive an initial HTTPS POST that Magento sends when the merchant activates integration. This post contains the Magento store URL, an `oauth_verifier`, the OAuth consumer key, and the OAuth consumer secret. The consumer key and secret are generated when the integration is created.

* Ask for a request token. A request token is a temporary token that the user exchanges for an access token. Use the following API to get a request token from Magento:

  `POST /oauth/token/request`

  See [Get a request token]( {{ page.baseurl }}/get-started/authentication/gs-authentication-oauth.html#pre-auth-token ) for more details about this call.

* Parse the request token response. The response contains an `oauth_token` and `oauth_token_secret`.

* Ask for a access token. The request token must be exchanged for an access token. Use the following API to get a request token from Magento:

  `POST /oauth/token/access`

  See [Get an access token]( {{ page.baseurl }}/get-started/authentication/gs-authentication-oauth.html#get-access-token ) for more details about this call.

* Parse the access token response. The response contains an `oauth_token` and `oauth_token_secret`. These values will be different than those provided in the request token response.

* Save the access token and other OAuth parameters. The access token and OAuth parameters must be specified in the `Authorization` header in each call to Magento.

## Related Topics

- [Web API authentication]({{ page.baseurl }}/get-started/authentication/gs-authentication.html)
- [OAuth-based authentication]( {{ page.baseurl }}/get-started/authentication/gs-authentication-oauth.html )
- [Magento System Requirements]({{ page.baseurl }}/install-gde/system-requirements.html)
- [Create the module file structure]({{ page.baseurl }}/extension-dev-guide/build/module-file-structure.html)
- [Create a component]({{ page.baseurl }}/extension-dev-guide/build/create_component.html)
