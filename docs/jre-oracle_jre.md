# Oracle JRE
The Oracle JRE provides Java runtimes from [Oracle][] project.  No versions of the JRE are available be default due to licensing restrictions.  Instead you will need to create a repository with the Oracle JREs in it and configure the buildpack to use that repository.  Unless otherwise configured, the version of Java that will be used is specified in [`config/oracle_jre.yml`][].

<table>
  <tr>
    <td><strong>Detection Criterion</strong></td>
    <td>Unconditional</td>
  </tr>
  <tr>
    <td><strong>Tags</strong></td>
    <td><tt>oracle=&lang;version&rang;</tt></td>
  </tr>
</table>
Tags are printed to standard output by the buildpack detect script

**NOTE:**  Unlike the [OpenJDK JRE][], this JRE does not connect to a pre-populated repository.  Instead you will need to create your own repository by:

1.  Downloading the Oracle JRE binary (in TAR format) to an HTTP-accesible location
1.  Uploading an `index.yml` file with a mapping from the version of the JRE to its location to the same HTTP-accessible location
1.  Configuring the [`config/oracle_jre.yml`][] file to point to the root of the repository holding both the index and JRE binary
1.  Configuring the [`config/components.yml`][] file to disable the OpenJDK JRE and enable the Oracle JRE

For details on the repository structure, see the [repository documentation][repositories].

## Configuration
For general information on configuring the buildpack, refer to [Configuration and Extension][].

The JRE can be configured by modifying the [`config/oracle_jre.yml`][] file.  The JRE uses the [`Repository` utility support][repositories] and so it supports the [version syntax][]  defined there.

| Name | Description
| ---- | -----------
| `repository_root` | The URL of the Oracle repository index ([details][repositories]).
| `version` | The version of Java runtime to use.  Candidate versions can be found in the the repository that you have created to house the JREs. Note: version 1.8.0 and higher require the `memory_sizes` and `memory_heuristics` mappings to specify `metaspace` rather than `permgen`.
| `memory_sizes` | Optional memory sizes, described below under "Memory Sizes".
| `memory_heuristics` | Default memory size weightings, described below under "Memory Weightings.

### Memory
The total available memory is specified when an application is pushed as part of it's configuration. The Java buildpack uses this value to control the JRE's use of various regions of memory. The JRE memory settings can be influenced by configuring the `memory_sizes` and/or `memory_heuristics` mappings.

Note: if the total available memory is scaled up or down, the Java buildpack does not re-calculate the JRE memory settings until the next time the appication is pushed.

#### Memory Sizes
The following optional properties may be specified in the `memory_sizes` mapping.

| Name | Description
| ---- | -----------
| `heap` | The maximum heap size to use. It may be a single value such as `64m` or a range of acceptable values such as `128m..256m`. It is used to calculate the value of the Java command line options `-Xmx` and `-Xms`.
| `metaspace` | The maximum Metaspace size to use. It is applicable to versions of Oracle from 1.8 onwards. It may be a single value such as `64m` or a range of acceptable values such as `128m..256m`. It is used to calculate the value of the Java command line options `-XX:MaxMetaspaceSize=` and `-XX:MetaspaceSize=`.
| `permgen` | The maximum PermGen size to use. It is applicable to versions of Oracle earlier than 1.8. It may be a single value such as `64m` or a range of acceptable values such as `128m..256m`. It is used to calculate the value of the Java command line options `-XX:MaxPermSize=` and `-XX:PermSize=`.
| `stack` | The stack size to use. It may be a single value such as `2m` or a range of acceptable values such as `2m..4m`. It is used to calculate the value of the Java command line option `-Xss`.
| `native` | The amount of memory to reserve for native memory allocation. It should normally be omitted or specified as a range with no upper bound such as `100m..`. It does not correspond to a switch on the Java command line.

Memory sizes together with _memory weightings_ (described in the next section) are used to calculate the amount of memory for each memory type. The calculation is described later.

Memory sizes consist of a non-negative integer followed by a unit (`k` for kilobytes, `m` for megabytes, `g` for gigabytes; the case is not significant). Only the memory size `0` may be specified without a unit.

The above memory size properties may be omitted, specified as a single value, or specified as a range. Ranges use the syntax `<lower bound>..<upper bound>`, although either bound may be omitted in which case the defaults of zero and the total available memory are used for the lower bound and upper bound, respectively. Examples of ranges are `100m..200m` (any value between 100 and 200 megabytes, inclusive) and `100m..` (any value greater than or equal to 100 megabytes).

Each form of memory size is equivalent to a range. Omitting a memory size is equivalent to specifying the range `0..`. Specifying a single value is equivalent to specifying the range with that value as both the lower and upper bound, for example `128m` is equivalent to the range `128m..128m`.

#### Memory Weightings
Memory weightings are configured in the `memory_heuristics` mapping of [`config/oracle_jre.yml`][]. Each weighting is a non-negative number and represents a proportion of the total available memory (represented by the sum of all the weightings). For example, the following weightings:

```yaml
memory_heuristics:
  heap: 15
  permgen: 5
  stack: 1
  native: 2
```

represent a maximum heap size three times as large as the maximum PermGen size, and so on.

Memory weightings are used together with memory ranges to calculate the amount of memory for each memory type, as follows.

#### Memory Calculation
The total available memory is allocated into heap, Metaspace or PermGen (depending on the version of Oracle), stack, and native memory types.

The total available memory is allocated to each memory type in proportion to its weighting. If the resultant size of a memory type lies outside its range, the size is constrained to
the range, the constrained size is excluded from the remaining memory, and no further calculation is required for the memory type. If the resultant size of a memory size lies within its range, the size is included in the remaining memory. The remaining memory is then allocated to the remaining memory types in a similar fashion. Allocation terminates when none of the sizes of the remaining memory types is constrained by the corresponding range.

Termination is guaranteed since there is a finite number of memory types and in each iteration either none of the remaining memory sizes is constrained by the corresponding range and allocation terminates or at least one memory size is constrained by the corresponding range and is omitted from the next iteration.

[`config/components.yml`]: ../config/components.yml
[`config/oracle_jre.yml`]: ../config/oracle_jre.yml
[Configuration and Extension]: ../README.md#configuration-and-extension
[OpenJDK JRE]: jre-open_jdk.md
[Oracle]: http://www.oracle.com/technetwork/java/index.html
[repositories]: extending-repositories.md
[version syntax]: extending-repositories.md#version-syntax-and-ordering
