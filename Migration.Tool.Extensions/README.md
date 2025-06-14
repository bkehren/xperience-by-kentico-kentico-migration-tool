# Migration Tool customization

To create custom migrations:

1. Create the custom migration class:
    - [Field migrations](#customize-field-migrations)
    - [Widget migrations](#customize-widget-migrations)
    - [Widget property migrations](#customize-widget-property-migrations)
    - [Page to widget migrations](#migrate-pages-to-widgets)
    - [Custom class mappings](#custom-class-mappings)
    - [Custom child links](#custom-child-links)
2. [Register the migration](#register-migrations)

## Customize field migrations

You can customize field migrations to customize the default mappings of fields of page types, modules, system objects, and forms. In the `Migration.Tool.Extensions/CommunityMigrations` folder, create a new file with a class that implements the `IFieldMigration` interface. Implement the following properties and methods required by the interface:

- `Rank` - An integer property that determines the order in which migrations are applied. Use a value lower than *100000*, as that is the value for system migrations.
- `ShallMigrate` - A boolean method that specifies whether the migration shall be applied for the current field. Use properties of the `FieldMigrationContext` object passed as an argument to the method to evaluate the condition. For each field, the migration with the lowest rank that returns `true` from the `ShallMigrate` method is used.
  - `SourceDataType` - A string property that specifies the [data type](https://docs.kentico.com/x/coJwCg) of the source field.
  - `SourceFormControl` - A string property that specifies the [form control](https://docs.kentico.com/x/lAyRBg) used by the source field.
  - `FieldName` - A string property that specifies the code name of the field.
  - `SourceObjectContext` - An interface that can be used to check the context of the source object (e.g. page vs. form).
- `MigrateFieldDefinition` - Migrate the definition of the field. Use the `XElement` attribute to transform the field.
- `MigrateValue` - Migrate the value of the field. Use the source value and the `FieldMigrationContext` with the same parameters as described in `ShallMigrate`.

You can see samples:

- [SampleTextMigration.cs](./CommunityMigrations/SampleTextMigration.cs)
  - A sample migration with further explanations
- [AssetMigration.cs](./DefaultMigrations/AssetMigration.cs)
  - An example of a usable migration

After implementing the migration, you need to [register the migration](#register-migrations) in the system.

## Customize widget migrations

You can customize widget migration to change the widget to which source widgets are migrated in the target instance. In the `Migration.Tool.Extensions/CommunityMigrations` folder, create a new file with a class that implements the `IWidgetMigration` interface. Implement the following properties and methods required by the interface:

- `Rank` - An integer property that determines the order in which migrations are applied. Use a value lower than *100000*, as that is the value for system migrations.
- `ShallMigrate` - A boolean method that specifies whether the migration shall be applied for the current widget. Use properties of the `WidgetMigrationContext` and `WidgetIdentifier` objects passed as an argument to the method to evaluate the condition. For each widget, the migration with the lowest rank that returns `true` from the `ShallMigrate` method is used.
  - `WidgetMigrationContext.SiteId` - An integer property that specifies the ID of the site on which the widget was used in the source instance.
  - `WidgetIdentifier.TypeIdentifier` - A string property that specifies the identifier of the widget.
- `MigrateWidget`- Migrate the widget data using the following properties:
  - `identifier` - A `WidgetIdentifier` object, see `ShallMigrate` to see properties.
  - `value` - A `JToken` object containing the deserialized value of the property.
  - `context` - A `WidgetMigrationContext` object, see `ShallMigrate` to see properties.

You can see a sample: [SampleWidgetMigration.cs](./CommunityMigrations/SampleWidgetMigration.cs)

After implementing the migration, you need to [register the migration](#register-migrations) in the system.

## Customize widget property migrations

In the `Migration.Tool.Extensions/CommunityMigrations` folder, create a new file with a class that implements the `IWidgetPropertyMigration` interface. Implement the following properties and methods required by the interface:

- `Rank` - An integer property that determines the order in which migrations are applied. Use a value lower than *100000*, as that is the value for system migrations.
- `ShallMigrate` - A boolean method that specifies whether the migration shall be applied for the current widget. Use properties of the `WidgetPropertyMigrationContext` and `propertyName` objects passed as an argument to the method to evaluate the condition. For each widget property, the migration with the lowest rank that returns `true` from the `ShallMigrate` method is used.
  - `WidgetPropertyMigrationContext.SiteId` - An integer property that specifies the ID of the site on which the widget was used in the source instance.
  - `WidgetPropertyMigrationContext.EditingFormControlModel` - An object representing the [form control](https://docs.kentico.com/x/lAyRBg) of the property.
  - `propertyName` - A string property that specifies the identifier of the property.
- `MigrateWidgetProperty`- Migrate the widget property data using the following properties:
  - `key` - Name of the property.
  - `value` - A `JToken` object containing the deserialized value of the property.
  - `context` - A `WidgetPropertyMigrationContext`. See `ShallMigrate` method to see the properties.

You can see samples:

- [Path selector migration](./DefaultMigrations/WidgetPathSelectorMigration.cs)
- [Page selector migration](./DefaultMigrations/WidgetPageSelectorMigration.cs)
- [File selector migration](./DefaultMigrations/WidgetFileMigration.cs)

After implementing the migration, you need to [register the migration](#register-migrations) in the system.

## Migrate pages to widgets

This migration allows you to migrate pages from the source instance as [widgets](https://docs.kentico.com/x/7gWiCQ) in the target instance. This migration can be used in the following ways:

- If you have a page with content stored in page fields, you can migrate the values of the fields into widget properties and display the content as a widget.
- If you have a page that serves as a listing and displays content from child pages, you can convert the child pages into widgets and as content items in the content hub, then link them from the widgets.

> :warning: The target page (with a [Page Builder editable area](https://docs.kentico.com/x/7AWiCQ)) and any [Page Builder components](https://docs.kentico.com/x/6QWiCQ) used in the migration need to be present in the system before you migrate content. The target page must be either the page itself or any ancestor of the page from which the content is migrated.

In `Migration.Tool.Extensions/CommunityMigrations`, create a new file with a class that inherits from the `ContentItemDirectorBase` class and override the `Direct(source, options)` method:

1. If the target page uses a [page template](https://docs.kentico.com/x/iInWCQ), ensure that the correct page template is applied.

    ```csharp
    // Store page uses a template and is the parent listing page
    if (source.SourceNode.SourceClassName == "Acme.Store")
    {
      // Ensures the page template is present in the system
      options.OverridePageTemplate("StorePageTemplate");
    }
    ```

2. Identify pages you want to migrate to widgets and use the `options.AsWidget()` action.

    ```csharp
    // Identifies pages by their content type
    else if (source.SourceNode.SourceClassName == "Acme.Coffee")
    {
        options.AsWidget("Acme.CoffeeSampleWidget", null, null, options =>
        {
            // Determines where to place the widget
            options.Location
                // Negative indexing is used - '-1' signifies direct parent node
                // Use the value of '0' if you want to target the page itself
                .OnAncestorPage(-1)
                .InEditableArea("main-area")
                .InSection("SingleColumnSection")
                .InFirstZone();

            // Specifies the widget's properties
            options.Properties.Fill(true, (itemProps, reusableItemGuid, childGuids) =>
            {
                // Simple way to achieve basic conversion of all properties, properties can be refined in the following steps
                var widgetProps = JObject.FromObject(itemProps);

                // The converted page is linked as a reusable content item into a single property of the widget.
                // NOTE: List the page class name app settings in ConvertClassesToContentHub to make it reusable!
                widgetProps["LinkedContent"] = LinkedItemPropertyValue(reusableItemGuid!.Value);

                // Link reusable content items created from page's original subnodes
                // NOTE: List the page class names in app settings in ConvertClassesToContentHub to make it reusable!
                widgetProps["LinkedChildren"] = LinkedItemsPropertyValue(childGuids);

                return widgetProps;
            });
        });
    }
    ```

You can see a sample: [SamplePageToWidgetDirector.cs](./CommunityMigrations/SamplePageToWidgetDirector.cs)

After implementing the content item director, you need to [register the director](#register-migrations) in the system.

## Register migrations

Register the migration in `Migration.Tool.Extensions/ServiceCollectionExtensions.cs` as a `Transient` dependency into the service collection:

- Field migrations - `services.AddTransient<IFieldMigration, MyFieldMigration>();`
- Widget migrations - `services.AddTransient<IWidgetMigration, MyWidgetMigration>();`
- Widget property migrations - `services.AddTransient<IWidgetPropertyMigration, MyWidgetPropertyMigration>();`
- Page to widget migrations - `services.AddTransient<ContentItemDirectorBase, MyPageToWidgetDirector>();`

## Custom class mappings

You can customize class mappings to adjust the content model between the source instance and the target Xperience by Kentico instance. For example, you can merge multiple page types into a single content type, remodel page types as [reusable field scehams](https://docs.kentico.com/x/remodel_page_types_as_reusable_field_schemas_guides), or migrate them to the [content hub](https://docs.kentico.com/x/barWCQ) as reusable content.

1. Create a new class.
2. Add an `IServiceCollection` extension method. Use a separate method for every class mapping that you wish to configure.
3. Within the extension method, define a new `MultiClassMapping` object:

    ```csharp
    var m = new MultiClassMapping(targetClassName, target =>
    {
      // Target content type name
      target.ClassName = "Acme.Event";
      // Database table name of the new type
      target.ClassTableName = "Acme_Event";
      // Display name of the content type
      target.ClassDisplayName = "My new transformed event";
      // Type of the class (specifies that the class is a content type)
      target.ClassType = ClassType.CONTENT_TYPE;
      // What the content type is used for (reusable content, pages, email, or headless)
      target.ClassContentTypeType = ClassContentTypeType.WEBSITE;
    });
    ```

4. Define a new primary key:

    ```csharp
    m.BuildField("EventID").AsPrimaryKey();
    ```

5. Define individual fields of the new content type:

    ```csharp
    // Builds a new title field
    var title = m.BuildField("Title");

    // You can map any number of source fields. The migration creates items of the target data class for every item from a mapped source data class.
    // The default migration is skipped for any data class / page type where you map at least one source field

    // Maps the "EventTitle" field from the source data class "_ET.Event1"
    // Sets the isTemplate parameter to true, which makes the new field inherit the source field definition as a template
    // The field definition is migrated according to the migration tool's data type and form control/component mappings
    title.SetFrom("_ET.Event1", "EventTitle", true);
    // Maps the "EventTitle" field from a second source data class "_ET.Event2"
    // The isTemplate parameter is not set, so only the value is taken from this source field
    title.SetFrom("_ET.Event2", "EventTitle");

    // You can modify the field definition, e.g., change the caption of the field
    title.WithFieldPatch(f => f.Caption = "Event title");
    ```

6. (*Optional*) You can add custom value conversions:

    ```csharp
    var startDate = m.BuildField("StartDate");
    startDate.SetFrom("_ET.Event1", "EventDateStart", true);
    
    // Uses value conversion to modify the field value
    startDate.ConvertFrom("_ET.Event2", "EventStartDateAsText", false,
        (v, context) =>
        {
            switch (context)
            {
                case ConvertorTreeNodeContext treeNodeContext:
                    // You can use the available TreeNode context here
                    // (var nodeGuid, int nodeSiteId, int? documentId, bool migratingFromVersionHistory) = treeNodeContext;
                    break;
                default:
                    // No context is available (possibly when the tool is extended with other conversion possibilities)
                    break;
            }

            return v?.ToString() is { } av && !string.IsNullOrWhiteSpace(av) ? DateTime.Parse(av) : null;
        });
    startDate.WithFieldPatch(f => f.Caption = "Event start date");
    ```

7. Register the class mapping to the dependency injection container:

    ```csharp
    serviceCollection.AddSingleton<IClassMapping>(m);
    ```

8. Ensure that your class mapping extension methods run during the startup of the migration tool. Call the methods from `UseCustomizations` in the [ServiceCollectionExtensions](/Migration.Tool.Extensions/ServiceCollectionExtensions.cs) class.

**Note**: Your mappings now replace the default migration functionality for all data classes (page types, custom tables or custom module classes) that you use as a source. Any class where you set at least one source field is affected. If you map only some fields from a source class, the remaining fields are not migrated at all.

### Remodel page types as reusable field schemas guide

For an end-to-end example of how to extract common fields from two page types from Kentico Xperience 13 and move them to a [reusable field schema](https://docs.kentico.com/x/D4_OD) shared by both web page content types in Xperience by Kentico follow this [migration guide](https://docs.kentico.com/x/remodel_page_types_as_reusable_field_schemas_guides) in the documentation.

Note that any usage of `ReusableSchemaBuilder` in custom class mappings cannot be combined together with the `Settings.CreateReusableFieldSchemaForClasses` configuration option.

### Sample class mappings

You can find sample class mappings in the [ClassMappingSample.cs](/Migration.Tool.Extensions/ClassMappings/ClassMappingSample.cs) file.

- `AddSimpleRemodelingSample` showcases how to change the mapping of a single page type
- `AddClassMergeSample` showcases how to merge two page types into a single content type
- `AddReusableRemodelingSample` showcases how to migrate a page type as reusable content


## Custom child links

This feature allows you to link child pages as referenced content items of a page converted to reusable content item.

This feature is available by means of content item director.

You can apply a simple general rule to link child pages e.g. in `Children` field or you can apply more elaborate rules. You can see samples of both approaches in [SampleChildLinkDirector.cs](./CommunityMigrations/SampleChildLinkDirector.cs)

After implementing the content item director, you need to [register the director](#register-migrations) in the system.


