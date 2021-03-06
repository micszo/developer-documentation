# Add content and edit views

To able to add and edit Content Item with our Field Type using PlatformUI, we need to create an edit view.
The content view will also be needed to display correct information when viewing the Content Item in the "Content structure" tab in PlatformUI.

## Implement edit view

### Template file

First, we need to add a template file that will be responsible for displaying an edit form:

```html
//Resources/public/templates/fields/edit/tweet.hbt

<div class="pure-g ez-editfield-row">
    <div class="pure-u ez-editfield-infos">
        <label for="ez-field-{{ content.contentId }}-{{ fieldDefinition.identifier }}">
            <p class="ez-fielddefinition-name">
                {{ translate_property fieldDefinition.names }}{{#if isRequired}}*{{/if}}:
            </p>
            <p class="ez-editfield-error-message">&nbsp;</p>
        </label>
    </div>
    <div class="pure-u ez-editfield-input-area ez-default-error-style">
        <div class="ez-editfield-input"><div class="ez-tweet-input-ui"><input type="url"
                value="{{ field.fieldValue.url }}"
                pattern="^https?:\/\/twitter.com\/([^\/]+)\/status\/[0-9]+$"
                class="ez-validated-input"
                id="ez-field-{{ content.contentId }}-{{ fieldDefinition.identifier }}"
                {{#if isRequired}} required{{/if}}
            ></div></div>
        {{> ez_fielddescription_tooltip }}
    </div>
</div>
```

### Javascript view file

Next, we need to implement Javascript file that will be responsible for validating and handling the form:

```javascript
//Resources/public/js/views/fields/ez-tweet-editview.js

YUI.add('ez-tweet-editview', function (Y) {
    "use strict";
    /**
     * Provides the field edit view for the Tweet (eztweet) fields
     *
     * @module ez-tweet-editview
     */
    Y.namespace('eZ');

    var FIELDTYPE_IDENTIFIER = 'eztweet';

    /**
     * Tweet edit view
     *
     * @namespace eZ
     * @class TweetEditView
     * @constructor
     * @extends eZ.FieldEditView
     */
    Y.eZ.TweetEditView = Y.Base.create('tweetEditView', Y.eZ.FieldEditView, [], {
        events: {
            '.ez-tweet-input-ui input': {
                'blur': 'validate',
                'valuechange': 'validate'
            }
        },

        /**
         * Validates the current input of tweet
         *
         * @method validate
         */
        validate: function () {
            var validity = this._getInputValidity();

            if ( validity.typeMismatch || validity.patternMismatch ) {
                this.set('errorStatus', Y.eZ.trans('url.not.valid', {}, 'fieldedit'));
            } else if ( validity.valueMissing ) {
                this.set('errorStatus', Y.eZ.trans('this.field.is.required', {}, 'fieldedit'));
            } else {
                this.set('errorStatus', false);
            }
        },

        /**
         * Defines the variables to be imported in the field edit template for tweet.
         *
         * @protected
         * @method _variables
         * @return {Object} containing isRequired
         * entries
         */
        _variables: function () {
            var def = this.get('fieldDefinition');

            return {
                "isRequired": def.isRequired
            };
        },

        /**
         * Returns the input validity state object for the input generated by
         * the tweet template
         *
         * See https://developer.mozilla.org/en-US/docs/Web/API/ValidityState
         *
         * @protected
         * @method _getInputValidity
         * @return {ValidityState}
         */
        _getInputValidity: function () {
            return this.get('container').one('.ez-tweet-input-ui input').get('validity');
        },

        /**
         * Returns the currently filled tweet value
         *
         * @protected
         * @method _getFieldValue
         * @return String
         */
        _getFieldValue: function () {
            return this.get('container').one('.ez-tweet-input-ui input').get('value');
        }
    });

    Y.eZ.FieldEditView.registerFieldEditView(
        FIELDTYPE_IDENTIFIER, Y.eZ.TweetEditView
    );
});
```

### Registering the view

We also have to create the YAML file where we will tell the system to use our files. The example configuration looks like this:

```yaml
//Resources/config/yui.yml

system:
    default:
        yui:
            modules:
                tweeteditview-ez-template:
                    type: 'template'
                    path: bundles/ezsystemstweetfieldtype/templates/fields/edit/tweet.hbt
                ez-tweet-editview:
                    requires: ['ez-fieldeditview', 'event-valuechange', 'tweeteditview-ez-template']
                    dependencyOf: ['ez-contenteditformview']
                    path: bundles/ezsystemstweetfieldtype/js/views/fields/ez-tweet-editview.js
```

We need to prepend this configuration, like we did with the template in the previous step. To do this, we have to edit dependency injection extension:

```php
// TweetFieldTypeBundle/DependencyInjection/EzSystemsTweetFieldTypeExtension.php

use Symfony\Component\DependencyInjection\Extension\PrependExtensionInterface;
use Symfony\Component\Yaml\Yaml;

class EzSystemsTweetFieldTypeExtension extends Extension implements PrependExtensionInterface
{
    public function prepend(ContainerBuilder $container)
    {
        $configDirectoryPath = __DIR__.'/../Resources/config';

        $this->prependYamlConfigFile($container, 'ez_platformui', $configDirectoryPath.'/yui.yml');
        $this->prependYamlConfigFile($container, 'ezpublish', $configDirectoryPath.'/ez_field_templates.yml');
    }

    private function prependYamlConfigFile(ContainerBuilder $container, $extensionName, $configFilePath)
    {
        $config = Yaml::parse(file_get_contents($configFilePath));
        $container->prependExtensionConfig($extensionName, $config);
    }
}
```

Remember to dump your assets before trying to run this code for the first time.
After this is done, we should be able to edit the Content Item containing our Tweet Field Type.

## Implement content view

The steps needed to create content view are analogical to the ones needed to create edit view.

### Template file

First, we need to add a template file that will be responsible for displaying the content of our Field Type:

```html
//Resources/public/templates/fields/view/tweet.hbt

<div class="ez-fieldview-row pure-g">
    <div class="ez-fieldview-label pure-u">
        <p class="ez-fieldview-name"><strong>{{ translate_property fieldDefinition.names }}</strong></p>
    </div>
    <div class="ez-fieldview-value pure-u"><div class="ez-fieldview-value-content">
        {{#if isEmpty }}
            {{translate 'fieldview.field.empty' 'fieldview'}}
        {{else}}
            {{ value.url }}
        {{/if}}
    </div></div>
</div>
```

### Javascript view file

Next, we need to implement Javascript file that will be responsible for handling the template:

```javascript
//Resources/public/js/views/fields/ez-tweet-view.js

YUI.add('ez-tweet-view', function (Y) {
    "use strict";
    /**
     * Provides the Tweet field view
     *
     * @module ez-tweet-view
     */
    Y.namespace('eZ');

    /**
     * The Tweet field view
     *
     * @namespace eZ
     * @class TweetView
     * @constructor
     * @extends eZ.FieldView
     */
    Y.eZ.TweetView = Y.Base.create('tweetView', Y.eZ.FieldView, [], {
        /**
         * Returns the value to be used in the template. If the value is not
         * filled, it returns undefined otherwise an object with a `url` entry.
         *
         * @method _getFieldValue
         * @protected
         * @return Object
         */
        _getFieldValue: function () {
            var value = this.get('field').fieldValue, res;

            if ( !value || !value.url ) {
                return res;
            }
            res = {url: value.url};

            return res;
        }
    });

    Y.eZ.FieldView.registerFieldView('eztweet', Y.eZ.TweetView);
});
```

### Registering the view

We also need to add information about our freshly created files to the already created configuration. To achieve this, let's add the following snippet to the `yui.yml` file:
```yaml
                tweetview-ez-template:
                    type: 'template'
                    path: bundles/ezsystemstweetfieldtype/templates/fields/view/tweet.hbt
                ez-tweet-view:
                    requires: ['ez-fieldview', 'tweetview-ez-template']
                    dependencyOf: ['ez-rawcontentview']
                    path: bundles/ezsystemstweetfieldtype/js/views/fields/ez-tweet-view.js
```

After these steps you should see correct information when viewing Content Item in "Content structure" tab in PlatformUI.

⬅ Previous: [Introduce a template](6_introduce_a_template.md)

Next: [Add a validation](8_add_a_validation.md) ➡
