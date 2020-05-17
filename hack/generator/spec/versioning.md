# Versioning



## Goals

**Auto-generate conversions:** As far as practical, we want to autogenerate the schema for each hub version along with the conversions to and from the actual ARM API versions. Hand coding all the required conversions doesn't scale across all the different Azure sevices, especially with the ongoing rate of change. 

**Allow for hand coded conversions:** While we expect to use code generation to handle the vast majority of needed conversions, we anticipate that some breaking API changes will require some of the conversion to be hand coded. We need to make it simple for these conversions to be introduced while still automatically generating the majority of the conversion.

**No modification of generated files.** Manual modification of generated files is a known antipattern that greatly increases the complexity and burden of updates. If some files have been manually changed, every difference showing after code generation needs to be manually reviewed before being committed. This is tediuos and error prone because the vast majority of auto generated changes will be perfectly fine. 

## Non-Goals

**Coverage of every case by code generation:** While it's likely that very high coverage will be achievable with code generation, we don't believe that it will be practical to handle every possible situation automatically. It's therefore necessary for the solution to have some form of extensibility allowing for the injection of hand written code.

## Proposed Solution

The "v1" storage version of each supported resource type will be created by merging all of the fields of all the distinct versions of the resource type, creating a *superset* type that includes every property declared across every version of the API.

* To maintain backward compatibility as Azure APIs evolve over time, we will include properties across all versions of the API, even for versions we are not currently generating as output. This ensures that properties in use by older APIs are still present and available for forward conversion to newer APIs, even as those older APIs age out of use.

TBC

## Issues

Open issues

* What if the type of a property changes?
* How do we handle extensibility?

Resolved issues:

* Do we merge all properties from all versions of an API? [Resolved]

### What if the type of a property changes?

What do we do if the type of a property is changed between two different versions of the API?

To illustrate, consider a meeting definition where start/end times of the meeting become strongly typed from one API version to the next:

``` go
package v20180101
type Meeting struct {
    Starts string
    Ends string
    // elided
}
```

``` go
package v20200101
type Meeting struct {
    Starts Time
    Ends Time
    // elided
}
```

We can also anticipate changes going in the other direction, where a strongly typed value becomes a **string** in order to provide greater flexibility.

#### Discussion

Do we define the type using `interface{}` to allow any value to be stored, but at the cost of type safety? What does this do to our CRD generation based on these types?

Can we reuse the work being done by @mattchr to handle OneOf schema types?

We need to ensure that the final solution is backward compatible when we already have copies of an object in storage and the type is changed in a new version. For example, if we already had a `Meeting` in storage and the new version made the type change detailed above, we can't retrospectively change the name of the properties to avoid name clashes.

#### Resolution

TBC

### Do we merge all properties from all versions of an API? [Resolved]

There are two potential approaches to take when defining the storage version of an object. We can merge just the properties defined on the versions of the API for which we are emitting support, or we can merge all the properties defined across all versions of the API.

#### Discussion

If we only merge properties from versions of the versions of the API for which we are emitting support, we run the risk of the storage API becoming incompatible with the course of time. To illustrate, consider the following definition for a `Person`:

``` go
package v20180101

type Person struct {
    FirstName string
    LastName  string
}
```

For the v20200101 version of the API, with different fields that work cleanly across a wider set of cultures:

``` go
package v20200101

type Person struct {
    FullName   string
    FamilyName string
    KnownAs    string
    SortKey    string
}
```

Assuming we are generating support for the most recent two versions of the API, we end up with a storage schema that looks like this:

``` go
package v1

type Person struct {
    FirstName string
    LastName  string
    FullName   string
    FamilyName string
    KnownAs    string
    SortKey    string
}
```

However, when a new version of the API is introduced, say v20210101 merging only the most recent two versions would give us the following storage schema:

``` go
package v1

type Person struct {
    FullName   string
    FamilyName string
    KnownAs    string
    SortKey    string
}
```

This would be incompatible with any existing versions in storage, breaking the backward compatibility promise we should be adherent to.

The problem occurs in different forms regardless of the number of supported versions being retained. A threshold of two versions was used above to keep the example constrained.

#### Resolution

The storage version will be generated by combining all available versions of the API to create a superset, regardless of the number of versions being emitted by the code generator.

### How do we handle extensibility?

How should we accomodate situations where the generated code needs to be augmented with some hand coded additions?

#### Discussion

We'll already be generating the required `ConvertTo()` and `ConvertFrom()` methods defined by the `Convertible` interface, as a part of supporting the usual hub-spoke type conversion model.

One possible approach would be to generate a custom interface for each type, with a name based on the ARM API being targetted

``` go 
package v20200101

type AssignablePerson {}
    AssignTo(person v1.Person) error 
    AssignFrom(person v1.Person) error
}
```

Our generated `ConvertTo()` and `ConvertFrom()` methods would test for implementation of this interface and call the appropriate methods at the end of the generated conversion steps:

``` go
package v20200101 

func (person Person) ConvertTo(p conversion.Hub) error {
    destination := p.(*v1.Person)
    // ... elided ...
    if assignable, ok := person.(AssignablePerson) {
        assignable.AssignTo(destination)
    }
}
```

If implemented, the necessary assignment code would live in parallel files next to the types being converted, allowing them to be regenerated easily.

#### Resolution

TBC


## See Also

[Hubs, spokes, and other wheel metaphors](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts.html)

## Glossary

**ARM:** Azure Resource Manager



## Definitions

 