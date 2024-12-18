# Resource Metadata Specification

Resources are indexed by Kubernetes name.
This led to a few useful traits provided by the underlying platform:

* Naming conflict resolution
* Forcing users to treat resources like cattle as names are immutable.
* Human readable resource names can be propagated as metadata to other components e.g. cloud provider tags, as they are guaranteed not to change

However there were also some drawbacks:

* Resource names are tied to Kubernetes (DNS) syntax e.g. 63 characters, a-z, 0-9 etc.
* Resources names are immutable, doing so would be akin to deletion and recreation (see the above point about cattle)

This proposal aims to provide a formal specification that allows the best user experience possible, while yielding simple and reusable software and also not impeding support processes to the extent that it's likely that mistakes will be made.

## Changelog

- v1.0.0 2024-05-29 (@spjmurray): Initial RFC
- v1.0.1 2024-06-06 (@spjmurray): Update to mirror reality
- v1.0.2 2024-07-17 (@spjmurray): Update ID and name documentation
- v1.0.3 2024-07-19 (@spjmurray): Add audit fields
- v1.0.4 2024-11-27 (@spjmurray): Add metadata tags

## Considerations

### Generic Metadata

Every resource in the system should have the following items, unless where specified.

#### Unique Identifier

The resource ID is a UUIDv4 (i.e. random) string.
As this directly maps to a Kubernetes resource name, it MUST be compatible with DNS labels.
As such it MUST start with a character.

#### Resource Name

This is a string that is attached to a resource as a label, for index performance.

Using labels limits the character set to that defined [by Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set).
Annotations on the other hand allow arbitrary text (e.g. it was used for a long time to store JSON for `kubectl apply`).
Annotations do however suggest the metadata is non-identifying, and cannot be used to index a resource, which makes resource ID _the_ de facto identifier at the API level.

#### Resource Description **OPTIONAL**

The description should be a verbose description of what the resource is for, useful where names have a specific schema, that is non-intuitive, and you want to give additional detail to viewers so they don't accidentally delete something.

#### Resource Tags **OPTIONAL**

Every resource should support metadata tags.
The idea here is that a resource can be consumed by higher level services, and those services may need to map some form of configuration to a resource for reconciliation.
Where names cannot be used to perform the mapping directly, tags offer a richer and less restrictive key-value store to apply generic metadata to a resource.

#### Creation and Deletion Time

Creation time is useful to see the age of a resource, and offers the ability to sort by age to better locate a resource in a client.

The deletion time is an indication that resource deletion has been requested, and thus other calls to edit or delete the resource should be inhibited to avoid false negative errors.

#### Auditing

Audit logs may be time-bound due to space constraints, so we should endeavor to keep track of the actor who is responsible for creating and modifying an object.
Additionally we need to record the modification time.

#### Provisioning Status

Typically this maps directly from a Kubernetes status condition, but to future proof things we should make it generic so we can add health monitoring to any resource trivially in future even if it doesn't have a Kubernetes controller.
This makes client handling far simpler.

By making this explicitly about provisioning, we can extend the metadata further with asynchronous health checks further down the line.

### Scoped Metadata

Historically, we've been lazy, in the sense that a GET against a collection endpoint returns everything that is in scope, typically within an organization.
This facilitates a global view of everything in the organization, particularly relevant to administration views.
Any more constrained view can be achieved by further scoping the data client-side by using the following metadata items.

#### Organization and Project

Presently, the organization is implicit as all relevant API resource endpoints are nested under an organization, but we can anticipate future views that take into consideration the entire platform e.g. to check an upgrade succeeded across all organizations.

The project allows resources to be grouped per-project for easy display of hierarchical constructs, and allowing roll-up of data into per project summaries an the client side.

Note that at present this is all keyed to the resource name, and will need to be migrated to unique IDs.

## Modeling Metadata

### GET Requests

Generic metadata should look like the following:

```yaml
components:
  schemas:
    resourceProvisioningStatus:
      type: string
      enum:
      - unknown
      - provisioning
      - provisioned
      - deprovisioning
      - error
    tag:
      type: object
      required:
      - name
      - value
      properties:
        name:
          type: string
        value:
          type: string
    tagList:
      type: array
      items:
        $ref: '#/components/schemas/tag'
    # Common metadata across reads/writes e.g. API mutable.
    resourceMetadata:
      type: object
      required:
      - name
      - description
      properties:
        name:
          type: string
        description:
          type: string
        tags:
          $ref: '#/components/schemas/tagList'
    # Things that every resource may have.
    staticResourceMetadata:
      type: object
      allOf:
      - $ref: '#/components/schemas/resourceMetadata'
      - type: object
        required:
        - id
        - creationTime
        properties:
          id:
            description: The unique resource ID.
            type: string
          creationTime:
            description: The time the resource was created.
            type: string
            format: date-time
          createdBy:
            description: The user who created the resource.
            type: string
          modifiedTime:
            description: The time a resource was updated.
            type: string
            format: date-time
          modifiedBy:
            description: The user who updated the resource.
            type: string
    # Things that only Kubernetes based resources may have.
    resourceReadMetadata:
      allOf:
      - $ref: '#/components/schemas/staticResourceMetadata'
      - type: object
        required:
        - provisioningStatus
        properties:
          deletionTime:
            type: string
            format: date-time
          provisioningStatus:
            $ref: '#/components/schemas/resourceProvisioningStatus'
```

Scoped metadata would look like the following.
Note the use of inheritance to create a hierarchy.

```yaml
components:
  schemas:
    organizationScopedResourceReadMetadata:
      allOf:
      - $ref: '#/components/schemas/resourceReadMetadata'
      - type: object
        required:
        - organizationId
        properties:
          organizationId:
            type: string
    projectScopedResourceReadMetadata:
      allOf:
      - $ref: '#/components/schemas/organizationScopedResourceReadMetadata'
      - type: object
        required:
        - projectId
        properties:
          projectId:
            type: string
```

A typical resource specific schema will look like:

```yaml
components:
  schemas:
    kubernetesclusterRead:
      type: object
      required:
      - metadata
      - spec
      properties:
        metadata:
          $ref: '#/components/schemas/projectScopedResourceReadMetadata'
        spec:
          # resource specific stuff goes here.
```

### POST and PUT Requests

Generic metadata should look like the following:

```yaml
components:
  schemas:
    resourceWriteMetadata:
      $ref: '#/components/schemas/resourceMetadata'
```

This quite clearly makes a distinction between mutable fields that can be written by a client, and read-only fields that are controlled by the server.

Through the magic of duck-typing, `resourceReadMetadata` *could* be used directly as a `resourceWriteMetadata`, although that will just result in the server rejecting the request as it doesn't conform to the schema.

A typical resource specific schema will look like:

```yaml
components:
  schemas:
    kubernetesclusterWrite:
      type: object
      required:
      - metadata
      - spec
      properties:
        metadata:
          $ref: '#/components/schemas/resourceWriteMetadata'
        spec:
          # resource specific stuff goes here.
```

From a client viewpoint, as regards PUT update operations, the metadata needs to be converted from the read version to the write version, then mutated as necessary, the specification can be copied verbatim.

## CI/CD Driver Considerations

When creating applications, the CI/CD driver typically creates a composite key of `${organization}/${project}/${resource}/${application}` to yield a unique fully qualified name for the application and other primitives like remote clusters.

These typically appear as either a name directly, or as labels that are indexed and can be queried in the CD tooling.
As these fields are presently resource names, we can very quickly locate a specific troublesome application based on the organization, project and resource (e.g. cluster) name.

Sadly as names are mutable now, we lose this ability as the fully qualified name would be composed of resource IDs, making lookups in the event of a support request both painfully slow and prone to human error as you have to translate between a logical name and a physical one.

We *could* propagate contextual naming information to resources, and then on to applications to restore this functionality, but that is added complexity, and will cause unnecessary reconciliation for simple metadata changes.
See the next section for further arguments.

## Cloud Platform Considerations

Like CI/CD drivers, we are able to tag cloud infrastructure with metadata that accelerates problem resolution time and reduces maintenance overheads.

Unlike applications, that can be annotated with labels, the tags attached to infrastructure are immutable and implemented by third party applications e.g. Cluster API.
Labels can be changed, but doing so would require a rolling rebuild of the infrastructure to facilitate the change.

Based on this observation, our common denominator is the use of IDs throughout the system.
Operations and support teams will just need to stomach the pain and associated risk that comes with the change.
