
## Side loading a local extension during the test session

You can use MsPortalFx-Test to write end to end tests that side load your local extension in the Portal. You can do this by specifying additional options in the Portal object. If you have not done so, please take a look at the *Installation* section of [this page](https://auxdocs.azurewebsites.net/en-us/documentation/articles/portalfx-testing-getting-started) to learn how to get started with MsPortalFx-Test. 

We'll write a test that verifies that the Browse experience for our extension has been correctly implemented. But before doing that we should have an extension to test and something to browse to, so let's work on those first.

To prepare the target extension and resource:

1. Create a new Portal extension in Visual Studio following [these steps](https://auxdocs.azurewebsites.net/en-us/documentation/articles/portalfx-creating-extensions) and then click CTRL+F5 to get it up and running. For the purpose of this example we named the extension 'LocalExtension' and we made it run in the default [https://localhost:44300](https://localhost:44300) address. 

1. That should have taken you to the Portal, so sign in and then go to New --> Marketplace --> Local Development --> LocalExtension --> Create.

1. In the **My Resource** blade, enter **theresource** as the resource name, complete the required fields and click the  Create button.

1. Wait for the resource to get created.

To write a test verifies the Browse experience while side loading your local extension:

1. Create a new TypeScript file called **localextensiontests.ts**.
 
1. In the created file, import the MsPortalFx-Test module and layout the Mocha test:

	```ts
	/// <reference path="./typings/index.d.ts" />
	
	import assert = require('assert');
	import testFx = require('MsPortalFx-Test');
	import until = testFx.until;
	
	describe('Local Extension Tests', function () {
		this.timeout(0);
	
		it('Can Browse To The Resource Blade', () => {
			
		});
	});
	```

1. In the **Can Browse To The Resource Blade** test body,  specify the credentials for the test session (replace with your actual Azure credentials):
 
	```ts
	// Hardcoding credentials to simplify the example, but you should never hardcode credentials
	testFx.portal.portalContext.signInEmail = 'johndoe@outlook.com';
	testFx.portal.portalContext.signInPassword = '12345';
	```

1. Now, use the **features** option of the portal.PortalContext object to enable the **canmodifyextensions** feature flag and use the **testExtensions** option to specify the name and address of the local extension to load:
 
	```ts
	testFx.portal.portalContext.features = [{ name: "feature.canmodifyextensions", value: "true" }];
	testFx.portal.portalContext.testExtensions = [{ name: 'LocalExtension', uri: 'https://localhost:44300/' }];
	```

1. Let's also declare a variable with the name of the resource that the test will browse to:

	```ts
	let resourceName = 'theresource';
	```
 
1. To be able to open the browse blade for our resource we'll need to know three things: The resource provider, the resource type and the title of the blade. You can get all that info from the Browse PDL implementation of your extension. In this case the resource provider is **Microsoft.PortalSdk**, the resource type is **rootResources** and the browse blade title is **My Resources**. With that info we can call the **openBrowseBlade** function of the Portal object:

	```ts
	return testFx.portal.openBrowseBlade('Microsoft.PortalSdk', 'rootResources', 'My Resources')
	```
 
1. From there on we can use the returned Blade object to filter the list, verify that only one row remains after filtering and select that row:
	
	```ts
	.then((blade) => {
        return testFx.portal.wait<testFx.Controls.GridRow>(() => {
            return blade.grid.rows.count().then((count) => {
                return count === 1 ? blade.grid.rows.first() : null;
            });
        }, null, "Expected only one row matching '" + resourceName + "'.");
    }).then((row) => {
        return row.click();				
	})
	```

1. And finally we'll verify the correct blade opened and will close the Portal when done:

	```ts
	.then(() => {
		let summaryBlade = testFx.portal.blade({ title: resourceName });
		return testFx.portal.wait(until.isPresent(summaryBlade));
	}).then(() => {
		return testFx.portal.quit();
	});
	```

1. Here for the complete test:

    ```ts
    /// <reference path="./typings/index.d.ts" />

    import assert = require('assert');
    import testFx = require('MsPortalFx-Test');
    import until = testFx.until;

    describe('Local Extension Tests', function () {
        this.timeout(0);

        it('Can Browse To The Resource Blade', () => {
            // Hardcoding credentials to simplify the example, but you should never hardcode credentials
            testFx.portal.portalContext.signInEmail = 'johndoe@outlook.com';
            testFx.portal.portalContext.signInPassword = '12345';
            testFx.portal.portalContext.features = [{ name: "feature.canmodifyextensions", value: "true" }];
            testFx.portal.portalContext.testExtensions = [{ name: 'LocalExtension', uri: 'https://localhost:44300/' }];
            
            let resourceName = 'theresource';
            
            return testFx.portal.openBrowseBlade('Microsoft.PortalSdk', 'rootResources', 'My Resources').then((blade) => {
                return blade.filterItems(resourceName);
            }).then((blade) => {
                return testFx.portal.wait<testFx.Controls.GridRow>(() => {
                    return blade.grid.rows.count().then((count) => {
                        return count === 1 ? blade.grid.rows.first() : null;
                    });
                }, null, "Expected only one row matching '" + resourceName + "'.");
            }).then((row) => {
                return row.click();				
            }).then(() => {
                let summaryBlade = testFx.portal.blade({ title: resourceName });
                return testFx.portal.wait(until.isPresent(summaryBlade));
            }).then(() => {
                return testFx.portal.quit();
            });
        });
    });
    ```

To add the required configuration and run the test:

1. Create a file named **config.json** next to localextensiontests.ts. Paste this in the file:

	```json
	{
	  "capabilities": {
	    "browserName": "chrome"
	  },
	  "portalUrl": "https://portal.azure.com"
	}
	```		

1. Compile your TypeScript test file:

		tsc localextensiontests.ts --module commonjs

1. Run Mocha against the generated JavaScript file:

		node_modules\.bin\mocha localextensiontests.js

The following output will be sent to your console as the test progresses:

	  Local Extension Tests
	Opening the Browse blade for the Microsoft.PortalSdk/rootResources resource type...
	Starting the ChromeDriver process...
	Performing SignIn...
	Waiting for the Portal...
	Waiting for the splash screen...
	Allowing trusted extensions...
	Waiting for the splash screen to go away...
	Applying filter 'theresource'...
	    √ Can Browse To The Resource Blade (22872ms)
	
	
	  1 passing (23s)