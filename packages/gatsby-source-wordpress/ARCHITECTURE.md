This doc goes over the high level areas of this plugin, roughly what they do, how they're related, and where to find the corresponding code.

- [Historic Info](#historic-info)
- [`gatsby-node.ts` Steps](#gatsby-nodets-steps)
- [Query Generation / Remote Schema Ingestion](#query-generation--remote-schema-ingestion)
  - [Good to know](#good-to-know)
- [Schema Customization](#schema-customization)
- [Interactions between Schema Customization and Query Generation](#interactions-between-schema-customization-and-query-generation)
  - [Fetched Types are Queryable Types](#fetched-types-are-queryable-types)
  - [Automatic Field Prefixing](#automatic-field-prefixing)
  - [Schema Caching](#schema-caching)
- [Sourcing nodes](#sourcing-nodes)
- [Preview](#preview)
- [Compatibility api (DX and security-ish feature)](#compatibility-api-dx-and-security-ish-feature)
- [We're using rematch which is a redux wrapper](#were-using-rematch-which-is-a-redux-wrapper)
- [Gatsby node api helpers/actions are stored in local redux (not gatsby redux)](#gatsby-node-api-helpersactions-are-stored-in-local-redux-not-gatsby-redux)
- [Caching](#caching)
- [Hard caching](#hard-caching)
- [Image processing](#image-processing)
- [HTML processing](#html-processing)
- [Basic Auth](#basic-auth)
- [Build caching & inc builds](#build-caching--inc-builds)
- [Debugging options](#debugging-options)
- [Plugin options schema and documentation generation](#plugin-options-schema-and-documentation-generation)
- [`gatsby develop` DX features](#gatsby-develop-dx-features)
- [Gatsby Node API's in use](#gatsby-node-apis-in-use)

## Historic Info

This plugin was a ground-up rewrite of `gatsby-source-wordpress@^3.0.0` which used the WordPress REST API to source data. `gatsby-source-wordpress@^4.0.0` uses WPGraphQL to source data and is an entirely different plugin with no shared code. Work for `^4.0.0` was initially done in the [`gatsby-source-wordpress-experimental` repo](https://github.com/gatsbyjs/gatsby-source-wordpress-experimental) before being moved to the Gatsby monorepo.

Inspired by this plugin, the [Gatsby GraphQL Toolkit](https://github.com/gatsbyjs/gatsby-graphql-toolkit) was written to make creating source plugins for GraphQL API's easier. Ideally this plugin would use the toolkit but since it didn't exist when this plugin was created that wasn't possible. There are also features this plugin has which the toolkit doesn't (yet) have, so we would need to update the toolkit before making this plugin use it. This would be a large amount of effort for a small amount of returned value.

I initially wrote this plugin in JS and later it was partially ported to TS. As a result there are a mix of JS and TS files. When I was initially planning this plugin, the Gatsby core team collectively made a decision that Gatsby core should be written in TS and any plugins should be written in JS to keep community contributions from requiring the knowledge of TS. This plugin turned out to be one of the largest Gatsby plugin codebase's and I later decided it 100% needs TS. If you want to convert some JS files to TS please feel welcome to do so! We will accept TS conversion PR's.

The file you're reading was created many months after the first stable version of this plugin was released - so this is a non-exhaustive explanation of the architecture. If you notice something is missing, please take a crack at adding it here, or if you don't know much about the part you're documenting open a Github discussion so we can hash it out together.

- @TylerBarnes

## `gatsby-node.ts` Steps

In `src/gatsby-node.ts` a helper (`runApisInSteps`) is being used to run different "steps" of the codebase in order. Many parts of the codebase count on something else happening at an earlier point, so `runApisInSteps` is an easy way to visualize that.

This file is the entry point for 99.999% of the plugin (`src/gatsby-browser.ts` only imports 1 css file) so it's a good jumping off point for looking at different areas of the plugin.

## Query Generation / Remote Schema Ingestion

Before we can source data from WPGraphQL we need to generate GraphQL queries to source that data. We do that by making an introspection query to WPGraphQL and then using the response to generate queries using a custom query builder (`src/steps/ingest-remote-schema/build-queries-from-introspection`).

This logic runs during the Gatsby Node API `createSchemaCustomization`.

See `src/steps/ingest-remote-schema`.

### Good to know

- As we're aware of what will be a future Gatsby node, the queries can be generated to only fetch data we wont already have. Connections from one node to other nodes only fetch the id(s).
- When a field with a selection set is queried more than 1 time, a fragment is automatically generated for it to keep query size smaller.
- Currently we fetch connection id's on both sides (for example `User.posts[].id` as well as `Post.author.id`) which can result in some amount of overfetching.
- In the future we should add an API to mark some field as being able to be resolved entirely via Gatsby without fetching any data. Since we have `Post.author.id` we don't need to fetch `User.posts[].id` since we can query for all posts that have the current user node id as the author.
- Unnecessary overfetching also occurs as the plugin doesn't know which fields are being used in the Gatsby site, so we fetch all WPGQL data that's available. In `gatsby develop` this is necessary as we don't know what data the developer will want, but in `gatsby build` we know exactly what queries are being made.
- We should automatically fetch only the fields that are queried for in cold builds and data updates outside of `gatsby develop`.

## Schema Customization

Using the same introspection query we used in query generation, we use the remote schema to generate the WP/Gatsby schema to work with the Gatsby node model and within the parameters of any related plugin options. For many different types of fields, we have a field transformer (`src/steps/create-schema-customization/transform-fields`) which transforms the remote schema introspection for those fields into type/field definitions that Gatsby's schema customization layer understands. All resolvers happen on the Gatsby side as an automatic replication of what the resolvers in WPGraphQL are doing. This is only for fields with no input arguments. We currently don't (and probably always wont) have a way to automatically carry over input arguments from WPGraphQL.

This logic runs during the Gatsby Node API `createSchemaCustomization`.

See `src/steps/create-schema-customization`.

## Interactions between Schema Customization and Query Generation

Plugin options that impact both of these areas are under the [`schema`](https://github.com/gatsbyjs/gatsby/blob/master/packages/gatsby-source-wordpress/docs/plugin-options.md#schema) section.

### Fetched Types are Queryable Types

As queries are generated we store up a list of which types and fields have been queried. This list is used when customizing the Gatsby schema to omit any fields or types which weren't queried due to plugin options or internal logic.

### Automatic Field Prefixing

In the case that a Union or Interface type has multiple different types that contain the same fields, fields are auto-prefixed with the typename + fieldname during query generation. Later during schema customization, there is resolver logic to account for this situation. Without this, many interface and union types would be unqueryable as the fields are conflicting and GQL can't differentiate between them. `[Typename].[fieldName]` must always resolve to a single type (considering a union or interface as a single type).

### Schema Caching

Remote schema ingestion caches itself, diffing an md5 of the last remote schema it saw vs the current remote schema on each update (`src/steps/ingest-remote-schema/diff-schemas.js`). Anytime these are different it will re-generate all queries. When this happens, we will also re-run schema customization. In production this will cause the plugin to re-source all nodes, in development it will only update the schema and log out a warning to run `gatsby clean` if the schema update included a data change. Without this each produciton data update would be 10 - 30 seconds slower and development schema changes could cause the plugin to constantly re-source all data resulting in 5min+ wait times each time you modified your WPGraphQL schema.

## Sourcing nodes

- options (limit, etc)

## Preview

## Compatibility api (DX and security-ish feature)

## We're using rematch which is a redux wrapper

## Gatsby node api helpers/actions are stored in local redux (not gatsby redux)

## Caching

## Hard caching

## Image processing

- lazyNodes
- retry on end of queue

## HTML processing

- links
- files

## Basic Auth

## Build caching & inc builds

## Debugging options

## Plugin options schema and documentation generation

## `gatsby develop` DX features

## Gatsby Node API's in use

See `src/gatsby-node.ts`

- onPreInit
- pluginOptionsSchema
- createSchemaCustomization
- sourceNodes
- onPreExtractQueries
- onPostBuild
- onCreatePage
- onCreateDevServer