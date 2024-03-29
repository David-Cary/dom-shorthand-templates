# DOM Shorthand Templates
This combines the [DOM Shorthand](https://www.npmjs.com/package/dom-shorthand) and [Keyed Value Templates](https://www.npmjs.com/package/keyed-value-templates) libraries to let you create DOM nodes from object templates.  This gives the advantages of being able to store and process such templates and their immediate results as JSON objects.

# Quickstart
## Installation
You can install this library though npm like so:
```
$ npm install --save dom-shorthand-templates
```

## Usage
The simplest way to get started with this is simply creating a DOMTemplateRenderer and calling it's renderTemplate function with the template and a rendering context, like so:
```
import { DOMTemplateRenderer } from 'dom-shorthand-templates'

const renderer = new DOMTemplateRenderer()
const node = renderer.renderTemplate(
  {
    tag: 'p',
    content: [
      {
        $use: '+',
        args: [
          "Hi ",
          {
            $use: 'get',
            path: ['name']
          },
          "!"
        ]
      }
    ]
  },
  {
    name: "Bo"
  }
)
```

This accepts a KeyedTemplateResolver as an optional contructor parameter, letting you specify custom resolver behavior if desired.  By default the resolver uses the DEFAULT_DIRECTIVES from the keyed value templates library.

Note that the return value for this is DOM Node.  If you want to preview the compiled template before it's converted to a node, use the `resolveTemplate` function instead with the same parameters.

### Context Renderers
If you want an easy way to reuse the same template with different contexts, try the `ContextRenderer`.  Such renderers accept a template as it's first constructor parameter and have a `renderContext` function that will resolve that template using the provided context, like so:
```
import { ContextRenderer } from 'dom-shorthand-templates'

const renderer = new ContextRenderer(
  {
    tag: 'span',
    content: [
      {
        $use: '+',
        args: [
          "Hi. My name is ",
          {
            $use: 'get',
            path: ['name']
          },
          "."
        ]
      }
    ]
  }
)
const nameTags = [
  renderer.renderContext({ name: "Jack" }),
  renderer.renderContext({ name: "Jill" })
]
```

As with basic template renderers, there's a `resolveContextView` function if you want to get the resolved template before it's converted to a node.

### Data Renderers
If you want to a step further and have certain context values fixed for each render you can use the `DataRenderer` subclass.  Such renderers let you specify a base context as their second constructor parameter.  That context will then be used to populate the context used by the `renderData` and it's reolved template counterpart `resolveDataView`.

For example, let's say you wanted to use some javascript math functions in your template.  You could do so like this:
```
import { DataRenderer } from 'dom-shorthand-templates'

const renderer = new DataRenderer(
  {
    tag: 'span',
    content: [
      {
        $use: 'get',
        path: [
          'Math',
          {
            name: 'round',
            args: [
              {
                $use: 'get',
                path: [
                  'data',
                  'value'
                ]
              }
            ]
          }
        ]
      }
    ]
  },
  { Math }
)
const node = render.renderData({ value: 1.4 })
```

As you may have noticed, the target value was added to the context's "data" property.  That's controlled by the renderer'a `dataKey` property, which can be set in the constructor by passing it in as the 3rd parameter.  (4th parameter is the resolver.)  If that key is an empty string, all data values will compied straight to the context as per Object.assign.

## Advanced Tricks
Here are few extra things you can do with these classes.

### Element References
If you add the template to the render context said template can reference it's own elements.  For example, lets say you had something like this:
```
    const template = {
      tag: 'div',
      content: [
        {
          tag: 'img',
          attributes: {
            src: 'https://some.site/img_01.gif'
          }
        },
        " equals ",
        {
          $use: 'get',
          path: [
            'template',
            'content',
            0
          ]
        }
      ]
    }
    const context = { template }
```

That would replicate the same image in two different spots within the div.  While you could do that manually, the upside is editing the original's attributes will be automatically applied to every duplicate.

You can also use these references to set up calculated values at other points in the document, like this:
```
    const template = {
      tag: 'div',
      content: [
        'a = ',
        '3',
        ", b = ",
        '4',
        ", a + b = ",
        {
          $use: '+',
          args: [
            {
              $use: 'cast',
              as: 'number',
              value: {
                $use: 'get',
                path: [
                  'template',
                  'content',
                  1
                ]
              }
            },
            {
              $use: 'cast',
              as: 'number',
              value: {
                $use: 'get',
                path: [
                  'template',
                  'content',
                  3
                ]
              }
            }
          ]
        }
      ]
    }
```

As with the replication trick, this has the benefit of automatically updating all those derived field for you should you need to change one of the variables in the future.  You can acheive a similar effect by making those variables part of the render context, but that means needing to store those values separately to get the same output each time.

### Components
You may have noticed the above replication trick will give you an exact copy of the original.  If you want to reuse an element but change some of it's attributes or content in each location, you'd need something like this:
```
    const template = {
      tag: 'div',
      content: [
        {
          $use: 'run',
          steps: [
            {
              $use: 'set',
              path: ['n'],
              value: 1
            },
            {
              $use: 'return',
              value: {
                $use: '+',
                args: [
                  '#',
                  {
                    $use: 'getVar',
                    path: ['n']
                  }
                ]
              }
            }
          ]
        },
        ', ',
        {
          $use: 'run',
          steps: [
            {
              $use: 'set',
              path: ['n'],
              value: 2
            },
            {
              $use: 'return',
              value: {
                $use: 'resolve',
                value: {
                  $use: 'get',
                  path: [
                    'template',
                    'content',
                    0,
                    'steps',
                    1,
                    'value'
                  ]
                }
              }
            }
          ]
        }
      ]
    }
```

Putting it inside a "run" directive lets you set up local variables for each location.  Meanwhile, the "resolve" directive ensures the copied sub-template gets resolved instead of just being copied and used as is.

As of version 1.1.0, you can take advantage of the `DataViewDirective` (keyed as "present") to do this a bit more compactly, like so:

```
    const template = {
      tag: 'div',
      content: [
        {
          $use: 'present',
          template: {
            $use: 'value',
            value: {
              $use: '+',
              args: [
                '#',
                {
                  $use: 'coalesce',
                  args: [
                    {
                      $use: 'getVar',
                      path: ['n']
                    },
                    '_NaN_'
                  ]
                }
              ]
            }
          }
        },
        ', ',
        {
          $use: 'present',
          data: {
            n: 1
          },
          template: {
            $use: 'get',
            path: [
              'template',
              'content',
              0,
              'template',
              'value'
            ]
          }
        }
      ]
    }
```

Note that the above example also shows how you can provide placeholder values using the "coalesce" directive.  Such directives use the first non-nullish value in their arguments list, so if you get the first argumnet a "getVar" directive and the second to a placeholder value that placeholder will only be used if the target variable is null or undefined.

Version 1.2.0 adds the `ContentDuplicationDirective`.  This function much like the DataViewDirective, but will clear ids attributes in the proces, ensuring this duplication doesn't result in multiple elements with the same id.  That directive uses the `source` property in place of `template`.  It also allows overwriting attribute values through the `attributes` property, should you want to assign an id to the newly created element.

### Values as Text

Version also 1.2.0 introduces the `ValueTextDirective` and `WrappedValueTextDirective` ditectives.  The former takes converts it's value property to text using the setting in it's options property.  The latter does much the same, save that it wraps the resulting text in an array.  Such wrapping means you can use that array as a templated elements content array.

The options objects for both directives can have the following values:
  * nullishText - Text to show when the value is null or undefined.  Normally set to an empty string when you don't want to see "undefined" and "null" strings.
  * stringQuote - Quotation marks to wrap around string values.
  * viaJSON - Indicates JSON stringify should be used to generate the text.
  * replacer - Parameter to passed on to any JSON stringify calls.
  * space - Either number of spaces or a set of characters to use for each indenting of a nested item.
  * depth - Level of indenting to apply to the object.  This is usually only used in recursive calls.

