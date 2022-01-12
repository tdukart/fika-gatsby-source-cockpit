# @tdukart/gatsby-source-cockpit

This is a fork of the apparently abandoned [@fika/gatsby-source-cockpit](https://github.com/fikaproductions/fika-gatsby-source-cockpit) to introduce support for Gatsby 4.x and fix a few bugs.

NOTE: This fork is a side project of a single person and is likely buggy. I'll do my best to work out bugs. Contributions are welcome!

NOTE 2: I am not a Fika Productions employee. There are a few mentions of their trademarks in here, but are intended as reference.

## Installation

```
npm install --save @tdukart/gatsby-source-cockpit
```

This project has `gatsby-source-filesystem` as a peer dependency, so don't forget to install it as well. (Assuming you already have Gatsby and React.)

```
npm install --save gatsby-source-filesystem
```

## Contributing

1. Fork main project on github [here](https://github.com/tdukart/fika-gatsby-source-cockpit).
2. Clone your fork.
3. Create a new branch on your local fork.
4. Commit and push your changes on this branch.
5. Create a pull request on the main project by going [here](https://github.com/tdukart/fika-gatsby-source-cockpit/compare), click on "compare across forks" and select your own branch in the "head fork" section.
6. Compare changes and submit pull request.

### Tips and tricks

While developing a GatsbyJS source plugin, it is useful to have a GatsbyJS project on the side in order to test it. To use the local plugin in your project instead of the one on NPM, you can run:

```
// In the plugin's folder
npm link

// In the GatsbyJS project's folder
npm link @tdukart/gatsby-source-cockpit
```

You'll have to install the plugin's peer dependencies in the plugin's folder as well (without saving them):

```
npm install --no-save gatsby react
```

Then, in order to unlink the local plugin and use the one from NPM again:

```
npm uninstall --no-save @tdukart/gatsby-source-cockpit
npm install
```

## How to use

Add this to your project `gatsby-config.js` file:

```
plugins: [
  {
    resolve: 'gatsby-source-filesystem',
    options: {
      name: 'src',
      path: `${__dirname}/src/`,
    },
  },
  {
    resolve: '@tdukart/gatsby-source-cockpit',
    options: {
      token: 'YOUR_COCKPIT_API_TOKEN',
      baseUrl:
        'YOUR_COCKPIT_API_BASE_URL', // (1)
      locales: ['EVERY_LANGUAGE_KEYS_DEFINED_IN_YOUR_COCKPIT_CONFIGURATION'], // (2)
      collections: [], // (3)
      singletons: [], // (4)
      aliases: {
        collection: {
          A_COLLECTION_NAME: 'AN_ALIAS',
          …
        },
        singleton: {
          A_SINGLETON_NAME: 'AN_ALIAS',
          …
        }
      }, // (5)
      brokenImageReplacement: 'AN_URL_TO_AN_IMAGE', // (6)
    },
  },
]
```

Notes:

1. E.g. `'http://localhost:8080'`.
2. E.g. `['en', 'fr']`.
3. The specific Cockpit collections you want to fetch. If empty or null all collections will be fetched. E.g. `['Products', 'Menu']`
4. Same as the `collections` parameter, but for the Cockpit singletons.
5. You can specify aliases for any Cockpit collection or singleton. Since it's not possible to have two GraphQL types with the same name in a schema, you can use this configuration to alias for instance a collection and a singleton sharing the same name (or with a difference of capitalization in the first character).  
   E.g. (for a singleton and a collection both named 'Team') `{ collection: { Team: 'Teams' } }`.
6. Replacement for broken image links. If `null`, the detected broken images will be removed. If an URL to an image, the broken image will be replaced with this image.

Adding the `gatsby-source-filesystem` dependency to your project grants access to the `publicURL` field resolver attribute on the file nodes that this plugin generates by extending the GraphQL type of the file nodes. So, as you can guess, the path specified in the plugin options could be anything, we do not need it to load any local files, we are just taking advantage of its extension of the file node type.

## How to query

Collections and singletons are converted into nodes. You can access many collection entries at once with this syntax:

(The collection is named 'team' or 'Team' in Cockpit.)

```
{
  allCockpitTeam(filter: { spiritAnimal: { eq: "tiger" } }) { // (1)
    edges {
      node { // (2)
        cockpitId // (3)
        cockpitCreated // (3)
        cockpitModified // (3)
        cockpitBy // (3)
        cockpitModifiedBy // (3)
        TeamMember1
        TeamMember2
        TeamMember3
        childrenCockpitTeam { ... } // (4)
      }
    }
  }
}
```

Notes:

1. You can filter amongst them.
2. Each node is a collection entry in an array.
3. You can get the original Cockpit element's id (aka the `_id`), creation and modification dates and authors' (ids for now) that way.
4. You can access descendant collection entries within that field if you have hierarchically structured your collection entries in Cockpit (_Custom sortable entries_ turned on).

Or you can access one entry at the time or a singleton that way:

(The collection is named 'definition' or 'Definition' in Cockpit.)

```
query($locale: String) { // (1)
    cockpitDefinition(cockpitId: { eq: "5bc78a3679ef0740297b4u04" }, lang: { eq: $locale }) { // (2)
        header {
            type
            value
        }
    }
}
```

Notes:

1. Using `query` with a name or not is optional in GraphQL. However, if you want to use variables from your page context, it is mandatory.
2. You can get the appropriate language by filtering on the `lang` attribute.

(The singleton is named 'vegetable' or 'Vegetable' in Cockpit.)

```
{
  cockpitVegetable(lang: { eq: "en" }) {
    category {
      type
      value
    }
  }
}
```

### Special types of Cockpit fields

#### Texts

Text fields with the option `{ "slug": true }` can access the slug that way:

```
{
  allCockpitBlogPost {
    edges {
      node {
        title {
          slug
        }
      }
    }
  }
}
```

#### Collection-Links

Collection-Link fields will see their value attribute refering to another or many others collection(s) node(s) (GraphQL foreign key). One to many Collection-Links are only supported for multiple entries of a single collection. This an example with a TeamMember collection entry linked within a Team collection:

```
{
  allCockpitTeam {
    edges {
      node {
        Header {
          type
          value
        }
        TeamMember {
          type // (1)
          value { // (2)
            id
            Name {
              value
            }
            Task {
              value
            }
          }
        }
      }
    }
  }
}
```

Notes:

1. The type is `'collectionlink'` and it was originally refering to an entry of the TeamMember collection.
2. The refered node is attached here. The language is preserved across these bindings.

#### Images and galleries

Image and gallery fields nested within a collection or singleton will be downloaded and will get one or more file(s) node(s) attached under the `value` attribute like this:

(You can then access the child(ren) node(s) a plugin like `gatsby-transformer-sharp` would create.)

```
{
  allCockpitTeamMember {
    edges {
      node {
        Portrait {
          value {
            publicURL // (1)
            childImageSharp {
              fluid {
                ...GatsbyImageSharpFluid
              }
            }
          }
        }
      }
    }
  }
}
```

Notes:

1. You can use this field to access your images if their formats are not supported by `gatsby-transformer-sharp` which is the case for `svg` and `gif` files.

#### Assets

Just like image fields, asset fields nested within a collection or singleton will be downloaded and will get a file node attached under the `value` attribute.

You can access the file regardless of its type (document, video, etc.) using the `publicURL` field resolver attribute on the file node.

#### Markdowns

Markdown fields nested within a collection or singleton will get a custom Markdown node attached under the `value` attribute. It mimics a file node — even if there is no existing Markdown file — in order to allow plugins like `gatsby-transformer-remark` to process them. Moreover, images and assets embedded into the Markdown are downloaded and their paths are updated accordingly. Example:

(You can then access the child node a plugin like `gatsby-transformer-remark` would create.)

```
{
  allCockpitDefinition {
    edges {
      node {
        Text {
          value {
            childMarkdownRemark {
              html
            }
            internal {
              content // (1)
            }
          }
        }
      }
    }
  }
}
```

Notes:

1. You can access the raw Markdown with this attribute.

#### Sets

The set field type allows to logically group a number of other fields together.

You can then access their values as an object in the `value` of the set field.

```
{
  allCockpitTeamMember {
    edges {
      node {
        ContactData { // field of type set
          value {
            telephone {
              value
            }
            fax {
              value
            }
            email {
              value
            }
          }
        }
      }
    }
  }
}
```

#### Repeaters

Repeater fields are one of the most powerful fields in Cockpit and allow support for two distinct use cases:

1.  Repeat any other field type (including the set type) an arbitrary number of times resulting in an array of fields with the same type (E.g. `[Image, Image, Image]`)
1.  Choose from a number of specified fields an arbitrary number of times resulting in an array where each entry might be of a different type (E.g. `[Image, Text, Set]`)

For the first case the values can be queried almost like a normal scalar field. The only difference is that two nested values are needed with the first one representing the array and the second one the value in the array.

```
{
  allCockpitTeamMember {
    edges {
      node {
        responsibilities { // field of type repeater
          value { // value of repeater (array)
            value // value of repeated field
          }
        }
      }
    }
  }
```

The second case is a bit more complicated - in order to not cause any GraphQL Schema conflicts each array value must be of the same type. To achieve this the `gatsby-source-cockpit` plugin implicitly wraps the values in a set field, generating one field in the set for each `fields` option supplied in the repeater configuration.

E.g. assuming the repeater field is configured with these options:

```
{
  "fields": [
    {
      "name": "title",
      "type": "text",
      "label": "Some text",
    },
    {
      "name": "photo",
      "type": "image",
      "label": "Funny Photo",
    }
  ]
}
```

then the following query is necessary to get the data:

```
{
  allCockpitTeamMember {
    edges {
      node {
        responsibilities { // field of type repeater
          value { // value of repeater (array)
            title {
              value
            }
            photo {
              value {
                ...
              }
            }
          }
        }
      }
    }
  }
```

**Note:** For this to work the fields specified in the `field` option need to have a `name` attribute which is not required by Cockpit itself. If the name attribute is not set, the plugin will print a warning to the console and generate a `name` value out of the value of the `label` attribute but it is recommended to explicitly specify the `name` value.

#### Layouts and layout-grids

The layout(-grid) field type allows to compose a view with UI components (buttons, divider, text, custom components, …).

You can access the whole components hierarchy using the `parsed` field. The type of each component is set in the `component` field. All the component settings defined in Cockpit are present in the `settings` field. Some of them are raw, others are processed:

- `class` is changed for `className`;
- `style` is processed into a CSS-in-JS-like object;
- `html` is accessible unchanged into `html`, but also accessible processed into `html_sanitize` (which is sanitized) and into `html_react` (which is a JS representation of a React component).

E.g.

```
{
  allCockpitPageLayout {
    edges {
      node {
        layout {
          value {
            parsed
          }
        }
      }
    }
  }
}
```

**Note:** Images and assets used in a layout aren't currently supported.

#### Objects

The object field type allows to structure data as a JSON object. You can get the whole object using the `data` field.

E.g.

```
{
  allCockpitLogEntries {
    edges {
      node {
        logEntry {
          value {
            data
          }
        }
      }
    }
  }
}
```

---

## Original plugin powered by &nbsp; — &nbsp;&nbsp; <a href="https://www.fikaproductions.com"><img align="center" width="200" height="200" src="src/images/logo.png"></a>
