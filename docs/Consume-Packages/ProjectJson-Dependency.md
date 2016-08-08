# Dependency Resolution in NuGet v3 / project.json 
 
For projects using project.json, NuGet has new rules for resolving the dependencies of the application. The most fundamental difference is that with package.config the resolution happens once at install time with various knobs to control the process and then the package graph is flattened out. Restore is merely downloading the packages. 

With NuGet3 and project.json restore is the process where the dependencies are discovered and conflicts are resolved. This happens both on install and on package restore. 

## Conflicts

Conflicts occurs when the same package id appears multiple times in the package graph, because two different packages depend on the same package id. 

When the ids are the same, but the version constraints are different, NuGet needs to pick a single version to be used by the application or the class library. In NuGet v2 this was both expensive and not always a deterministic process. NuGet v3 attempts to follow rules to achieve the following: 

1. Resolution is more predictable. 

2. Resolution is stable from build to build. 

3. Resolution is faster. 

There are four main rules that the dependency resolver applies: lowest applicable version, floating versions, nearest-wins, cousin dependencies. 

## Lowest applicable version  ##

NuGet restores the lowest possible version of a package as defined by its dependencies. This rule also applied to dependencies on the application or the class library unless declared as floating (see floating dependencies section for more details).  

![Figure 1](/images/consume/projectJson-dependency-1.png)

*Figure 1*

In Figure 1, 1.0-Beta is considered lower than 1.0, hence a minimal version of 1.0 is chosen.

![Figure 2](/images/consume/projectJson-dependency-2.png)

*Figure 2*

In Figure 2, the package version 2.1 is not available on the feed. But since the version constraint is >= 2.1 NuGet will pick the next lowest version it can find. In this case that is 2.2.

![Figure 3](/images/consume/projectJson-dependency-3.png)

*Figure 3*

In Figure 3 version 1.2 of Package A does not exist on the feed, but since the version constraint is an exact match NuGet will not give you the next lowest version (1.3) and instead an error will be generated.

## Floating Versions

When a floating version constraint is specified then NuGet will resolve the **highest** version of a package that matches the version pattern, for example 6.0.* will get the highest version of a package that starts with 6.0, see Figure 4 

![Figure 4](/images/consume/projectJson-dependency-4.png)

*Figure 4 - Floating Dependency*

You can put a floating dependency in your project.json, but not in your NuSpec. A package generated by the build must have a version constraint based on the version that is resolved during package restore. In the example in Figure 4 if MyApp is a class library that is producing a NuGet package then the NuSpec should have >= 6.0.1 as a dependency, as that is the version that was resolved and built against.

**Why floating dependencies? **Floating dependencies give you the ability to depend on whatever the latest version of a package is. For example, if your company is producing several NuGet packages that your application consumes then you likely want to depend on the latest of that package without having to update your app each time a new version comes out. This simplifies tremendously continuous integration scenarios.

## Nearest Wins
The dependency resolver prefers versions that are "closer" to the application, if they are declared on an ancestor of the dependency version constraint being ignored

![Figure 5](/images/consume/projectJson-dependency-5.png)

*Figure 5 - APPLICATION USING NEAREST WINS (Green is the picked version)*

In Figure 2, the green packages (A >= 1.0 and B >= 2.0) would be selected, while the grayed package (B >= 1.0) would be ignored. This is because a direct ancestor of "B >= 1.0" depends on a different version of B, and the resolver prefers that version.

**Why nearest wins?** Nearest-Wins allows a package/application to override any particular version of a package in its dependency tree by directly depending on the version of its choice. Note that downgrading produces an error today (we are considering turning it into a warning [https://github.com/NuGet/Home/issues/897](https://github.com/NuGet/Home/issues/897).

The other benefit is that it is a pruning mechanism for large dependency trees with many repeated nodes as we don't follow those repeated nodes. This makes a significant performance improvement with the size of graphs introduced by the BCL packages.

![Figure 6](/images/consume/projectJson-dependency-6.png)

*Figure 6 - APPLICATION USING NEAREST WINS (Green is the picked version)*

In Figure 2 package A is stating that it depends on Package C 2.0 even though B depends on Package C 1.0. Because A is taking a direct dependency on C, and is an ancestor of C, the nearest wins rule applies and the resolver will use C 2.0. 

## Cousin Dependencies ##

![Figure 7](/images/consume/projectJson-dependency-7.png)

*Figure 7 - Cousin Dependencies)*

In Figure 7 both Package A and C depend on Package B. But C is not an ancestor of "B 1.0" so the nearest wins rule does not apply. Instead the resolver uses the minimum acceptable version, which in this case is "B 2.0".

**When deciding between multiple cousin dependencies, the resolver use the lowest version that satisfies all version requirements.  See lowest application version and floating dependency rules.**

![Figure 8](/images/consume/projectJson-dependency-8.png)

*Figure 8 - Irreconcilable cousin dependencies)*

In Figure 8 both package A and package C depend on B, just as in Figure 7, but this time package A has an exact version constraint on version 1.0 whilst package C has a greater than 2.0 constraint. This is an error condition for the resolver as it cannot satisfy all of the version constraints. In order to fix it, the consumer needs to specify Package B's version as a top level dependency of MyApp, which will lead to a nearset win rule being applied.