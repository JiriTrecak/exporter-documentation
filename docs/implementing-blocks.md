# Implementing custom blocks

This guide will walk you through implementing custom block for an exporter. For the purpose of better understanding what is going on, we are using this exporter as example, which contains full definition of the block we will be implementing (dynamic health).

When building custom blocks, there are few things that work together. This guide touches all the topics needed to successfully deploy a new block:

- Declaration of custom block in `exporter.json`
- Enabling block for templating engine
- Writing appropriate template
- (optionally) extending block with global documentation-wide configuration
- Writing custom javascript functions for logic
- Deployment & Testing 

## What will we do?

We are going to build a dynamic health block that renders the banner similarly to how [Seeds design system does it](https://seeds.sproutsocial.com/components/banner/) - click "healthy". In order to do this, we will define a custom block, put that block into editor, and let exporter engine build the template for the block dynamically. 

The data for the block will come from the spreadsheet ([test spreadsheet here](https://docs.google.com/spreadsheets/d/1foch4-Dn59cyJlGh6FdfPLi-0-RtyrMg8FkbSAtrPCI/edit#gid=0)) and we will be able to identify component by providing attribute `id` inside the editor which is equal to `id` column inside that spreadsheet. This block will update everytime we will run export of the documentation (locally, or publish). Now let's get started.


## 1. Declaration of custom block in `exporter.json`
> Dynamic health example of `exporter.json` is [here](https://github.com/JiriTrecak/exporter-documentation/blob/master/exporter.json#L215).

In this file, you provide a new block definition. New block has several required attributes such as ID, name etc., and our VSCode extension will give you hints as to what all is needed. 

In addition to main metadata, it also defines properties that users have to fill in in the interface of the editor. Those properties you will then get as attributes of the block (more on that later). Please note that in order to have that block show and test, the exporter has to be installed in your workspace and selected as active (deployment & testing section).

### Note: Testing

We strongly recommend that you first configure the exporter.json and then immediately publish that change to your workspace, even without the actual implementation (html, js) of the block itself. This is because at that point you can spawn the block using `/health` in the documentation editor, and you will get the block and the data you've set inside the editor (like `componentId`) in VSCode as something you can debug that is now part of the page structure - making testing far easier. The block will just not render yet, as there is no associated template with it.

To test the exporter, simply go to `code integration > store > add (+)` and point to your forked directory. Then, go to documentation settings (gear icon) and switch `active exporter` to the exporter that you have created. Strongly recommend changing name in `exporter.json` so you know which one is which.

If you have used the dynamic option for configuring the spreadsheet information such as ID or Page Name (see `point 3 / getting documentation configuration` of this guide), don't forget to fill that information inside the documentation settings, otherwise you will get error message as per defined logic in that template.


## 2. Enabling block to be properly processed by templating engine
> Dynamic health example of `template that references the block` is [here](https://github.com/JiriTrecak/exporter-documentation/blob/master/src/page_body/structure/page_body_structure_block_custom.pr)

When you have created a definition, you will now be able to use that block through its ID. In the example above, you can see that I `switch` the block, and inject a new template, which defines the block frontend code. In the example above, I created template `page_block_custom_component_health_dynamic.pr` so I am injecting that:

```typescript
{[ const block = context /]}

{* Generate each block depending on this custom block identifier *}
{[ switch block.key ]}
{[ case "io.supernova.documentation-main.markdown" ]}
    {[ inject "page_block_custom_markdown" context block /]}
{[ case "io.supernova.documentation-main.component-health-dynamic" ]}         // Your block ID
    {[ inject "page_block_custom_component_health_dynamic" context block /]}  // Injected blueprint defining FE code of the block

{[/]}
```


## 3. Writing template for the dynamic health block
> Dynamic health example of `block template` is [here](https://github.com/JiriTrecak/exporter-documentation/blob/master/src/page_body/structure/blocks_custom/page_block_custom_component_health_dynamic.pr)

Now, every time you put this block into the editor, the blueprint you created will be invoked. The content can be any other template, so HTML code, HTML combined with javascript functions you declare in the engine or just about anything really. Let's disect the example code, with aditional comments added for clarity:

### Accessing block from context

Whatever you inject (inject [page] context [variable]) becomes available through `context` property, so we are sending the block down to access all its information:

```typescript
{[ const block = context /]}
```

### Getting block-defined properties

Each block has `properties` variable which is object containing key/value exactly as you have defined it with `exporter.json`. You will get either default value, or the value that was provided by the user (so in this case, user would say add "Button" as componentId in the editor and you'd get that here. All properties you define for that block are accessible through that object and will always be present.

```
{* This gets template}
{[ let componentId = block.properties.componentId /]}
```

In this case, we are accessing `componentId` which is `key` of the property defined in `exporter.json`. Inside the editor, the writer of the documentation can add any value, such as `Button`, which we will use later.

### Getting documentation configuration

In some cases, you'd like to set configuration that can be set on documentation-level, in case of dynamic health block, that would be the spreadsheet ID that provides the data, and also the name of the spreadsheet page to take the data from. You can either hardcode this, or you can use dynamic configuration system provided by the editor. To define configuration for those properties on level of documentation, modify your `exporter.json` with example code of [how such definition looks like](https://github.com/JiriTrecak/exporter-documentation/blob/master/exporter.json#L180).

Once defined, you can access the data fromt he configuration using top-level function `exportConfiguration()`:

```typescript
{[ let sourceIdentifier = exportConfiguration().componentHealthSourceIdentifier /]}
{[ let sourceName = exportConfiguration().componentHealthSourceName /]}
```

### Using custom javascript functionality

Finally, you might need custom javascript logic. Each javascript function you define is top-level, so you can invoke all your custom functions directly inside blueprints:

```typescript
// Create download sheet URL, and get its data 
{[ let sheetUrl = constructGoogleSheetCSVUrl(sourceIdentifier, sourceName) /]}
{[ let data = getNetworkData(sheetUrl) /]}

// Retrieve information from the API. To make it easy, we make block through javascript API that will have all required information
// We also gracefully handle any problem with retrieving the data
{[ if !data ]}
    <p>Unable to fetch information about component ID {{ componentId }}</p>
{[ else ]}
    // 
    {[ let dynamicBlock = constructDynamicHealthBlock(componentId, data) /]}
    {[ inject "page_block_custom_component_health" context dynamicBlock /]}
{[/]}
```

In the code above, we:

- Construct URL of spreadsheet with information we gotten from settings
- Get network data using pre-built helper function `getNetworkData` which does everything for you
- Create data structure that holds the information we gotten from the spreadsheet
- Inject blueprint that takes the data and renders UI elements (rendering could be put into one bluprint, but better to split it up)


## 4. Defining custom javascript

To define custom javascript function:

1. `cd` to `/typescript` folder and run `npm run watch` - this makes it so changes to typescript are compiled live using webpack and are immediately available to the exporter templates
2. Register new function so it is accessible by templating language (example [here](https://github.com/JiriTrecak/exporter-documentation/blob/master/typescript/src/index.ts)) using `Pulsar.registerFunction` in your ts entrypoint `index.ts`.
3. Write the function! Full health implementation of logic for this block can be found [here](https://github.com/JiriTrecak/exporter-documentation/blob/master/typescript/src/doc_functionality/health.ts)


## 5. Writing the rendering template

We have already build the main logic of the block, but we still don't have the template HTML representation. To do this, we will simply add another template file called `page_block_custom_component_health` and create HTML file that will represent the tag we want to show inside the documentation, and also the overlay that should get generated on click. You can find fully coded template which recreates [Seeds documentation here](https://github.com/JiriTrecak/exporter-documentation/blob/master/src/page_body/structure/blocks_custom/page_block_custom_component_health.pr)

### Adding CSS

Usually, blocks need CSS, so you can add that to `assets/css/stylesheet.css` or any other that you define yourself - just don't forget to add the link to load that CSS to `page_head_styles.pr`.

### Adding Assets

If you need any external assets, like icons etc., you can put them into `assets` directly, and then link to the asset by using the following syntax:

```
<link rel="stylesheet" type="text/css" href="{{ assetUrl("css/stylesheet.css", ds.documentationDomain()) }}" />
``` 


