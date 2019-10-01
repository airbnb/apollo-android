# Apollo GraphQL Client for Android

[![GitHub license](https://img.shields.io/badge/license-MIT-lightgrey.svg?maxAge=2592000)](https://raw.githubusercontent.com/apollographql/apollo-android/master/LICENSE) [![Get on Slack](https://img.shields.io/badge/slack-join-orange.svg)](http://www.apollostack.com/#slack)
[![Build status](https://travis-ci.org/apollographql/apollo-android.svg?branch=master)](https://travis-ci.org/apollographql/apollo-android)
[![GitHub release](https://img.shields.io/github/tag/apollographql/apollo-android.svg)](https://github.com/apollographql/apollo-android/releases)

Apollo-Android is a GraphQL compliant client that generates Java models from standard GraphQL queries.  These models give you a typesafe API to work with GraphQL servers.  Apollo will help you keep your GraphQL query statements together, organized, and easy to access from Java. Change a query and recompile your project - Apollo code gen will rebuild your data model.  Code generation also allows Apollo to read and unmarshal responses from the network without the need of any reflection (see example generated code below).  Future versions of Apollo-Android will also work with AutoValue and other value object generators.

## Adding Apollo to your Project

The latest Gradle plugin version is 0.3.3.

To use this plugin, add the dependency to your project's build.gradle file:

```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.apollographql.apollo:gradle-plugin:0.3.3'
    }
}
```

Latest development changes are available in Sonatype's snapshots repository:

```groovy
buildscript {
  repositories {
    jcenter()
    maven { url 'https://oss.sonatype.org/content/repositories/snapshots/' }
  }
  dependencies {
    classpath 'com.apollographql.apollo:gradle-plugin:0.4.0-SNAPSHOT'
  }
}
```

The plugin can then be applied as follows within your app module's `build.gradle` :

```
apply plugin: 'com.apollographql.android'
```

The Android Plugin must be applied before the Apollo plugin

### Kotlin

If using Apollo in your Kotlin project, make sure to apply the Apollo plugin before your Kotlin plugins within your app module's `build.gradle`:

```
apply plugin: 'com.apollographql.android'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
...
```

## Generate Code using Apollo

Follow these steps:

1) Put your GraphQL queries in a `.graphql` file. For the sample project in this repo you can find the graphql file at `apollo-sample/src/main/graphql/com/apollographql/apollo/sample/GithuntFeedQuery.graphql`. 

```
query FeedQuery($type: FeedType!, $limit: Int!) {
  feed(type: $type, limit: $limit) {
    comments {
      ...FeedCommentFragment
    }
    repository {
      ...RepositoryFragment
    }
    postedBy {
      login
    }
  }
}

fragment RepositoryFragment on Repository {
  name
  full_name
  owner {
    login
  }
}

fragment FeedCommentFragment on Comment {
  id
  postedBy {
    login
  }
  content
}
```

Note: There is nothing Android specific about this query, it can be shared with other GraphQL clients as well

2) You will also need to add a schema to the project. In the sample project you can find the schema `apollo-sample/src/main/graphql/com/apollographql/apollo/sample/schema.json`. 

You can find instructions to download your schema using apollo-codegen [HERE](http://dev.apollodata.com/ios/downloading-schema.html)

3) Compile your project to have Apollo generate the appropriate Java classes with nested classes for reading from the network response. In the sample project, a `FeedQuery` Java class is created here `apollo-sample/build/generated/source/apollo/com/apollographql/apollo/sample`.

Note: This is a file that Apollo generates and therefore should not be mutated.


## Consuming Code

You can use the generated classes to make requests to your GraphQL API.  Apollo includes an `ApolloClient` that allows you to edit networking options like pick the base url for your GraphQL Endpoint.

In our sample project, we have the base url pointing to `https://githunt-api.herokuapp.com/graphql`

There is also a #newCall instance method on ApolloClient that can take as input any Query or Mutation that you have generated using Apollo.

```java

apolloClient.newCall(FeedQuery.builder()
                .limit(10)
                .type(FeedType.HOT)
                .build()).enqueue(new ApolloCall.Callback<FeedQuery.Data>() {

            @Override public void onResponse(@Nonnull Response<FeedQuery.Data> dataResponse) {

                final StringBuffer buffer = new StringBuffer();
                for (FeedQuery.Data.Feed feed : dataResponse.data().feed()) {
                    buffer.append("name:" + feed.repository().fragments().repositoryFragment().name());
                    buffer.append(" owner: " + feed.repository().fragments().repositoryFragment().owner().login());
                    buffer.append(" postedBy: " + feed.postedBy().login());
                    buffer.append("\n~~~~~~~~~~~");
                    buffer.append("\n\n");
                }

				// onResponse returns on a background thread. If you want to make UI updates make sure they are done on the Main Thread.
                MainActivity.this.runOnUiThread(new Runnable() {
                    @Override public void run() {
                        TextView txtResponse = (TextView) findViewById(R.id.txtResponse);
                        txtResponse.setText(buffer.toString());
                    }
                });

            }

            @Override public void onFailure(@Nonnull Throwable t) {
                Log.e(TAG, t.getMessage(), t);
            }
        });
             
```

## Custom Scalar Types

Apollo supports Custom Scalar Types like `DateTime` for an example.

You first need to define the mapping in your build.gradle file. This will tell the compiler what type to use when generating the classes.

```gradle
apollo {
    customTypeMapping['DateTime'] = "java.util.Date"
    customTypeMapping['Currency'] = "java.math.BigDecimal"
}
```

Then register your custom adapter:

```java
CustomTypeAdapter<Date> customTypeAdapter = new CustomTypeAdapter<Date>() {
    @Override
    public Date decode(String value) {
        try {
            return ISO8601_DATE_FORMAT.parse(value);
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public String encode(Date value) {
        return ISO8601_DATE_FORMAT.format(value);
    }
};

// use on creating ApolloClient
ApolloClient.builder()
    .serverUrl(serverUrl)
    .okHttpClient(okHttpClient)
    .normalizedCache(normalizedCacheFactory, cacheKeyResolver)
    .addCustomTypeAdapter(CustomType.DATETIME, customTypeAdapter)
    .build();
```

## Support For Cached Responses

Apollo GraphQL client allows you to cache responses, making it suitable for use even while offline. The client can be configured with 3 levels of caching:

 - **HTTP Response Cache**: For caching raw http responses.
 - **Normalized Disk Cache**: Per node caching of responses in SQL. Persists normalized responses on disk so that they can used after process death. 
 - **Normalized InMemory Cache**: Optimized Guava memory cache for in memory caching as long as the App/Process is still alive.  

### Usage

Raw HTTP Response Cache:
```java
//Directory where cached responses will be stored
File file = new File("/cache/");

//Size in bytes of the cache
int size = 1024*1024;

//Strategy for deciding when the cache becomes stale
EvictionStrategy evictionStrategy = new TimeoutEvictionStrategy(5, TimeUnit.SECONDS);

//Create the http response cache store
ResponseCacheStore cacheStore = new DiskLruCacheStore(file, size); 

//Build the Apollo Client
ApolloClient apolloClient = ApolloClient.builder()
                                    .serverUrl("/")
                                    .httpCache(cacheStore, evictionStrategy)
                                    .okHttpClient(okHttpClient)
                                    .build();
```

Normalized Disk Cache:
```java
//Create the ApolloSqlHelper. Please note that if null is passed in as the name, you will get an in-memory SqlLite database that 
// will not persist across restarts of the app.
ApolloSqlHelper apolloSqlHelper = ApolloSqlHelper.create(context, "db_name");

//Create NormalizedCacheFactory
NormalizedCacheFactory cacheFactory = new SqlNormalizedCacheFactory(apolloSqlHelper);

//Create the cache key resolver
CacheKeyResolver<Map<String, Object>> resolver =  new CacheKeyResolver<Map<String, Object>>() {
          @Nonnull @Override public CacheKey resolve(@Nonnull Map<String, Object> objectSource) {
            String id = (String) objectSource.get("id");
            if (id == null || id.isEmpty()) {
              return CacheKey.NO_KEY;
            }
            return CacheKey.from(id);
          }
        }

//Build the Apollo Client
ApolloClient apolloClient = ApolloClient.builder()
                                    .serverUrl("/")
                                    .normalizedCache(cacheFactory, resolver)
                                    .okHttpClient(okHttpClient)
                                    .build();
```

Normalized InMemory Cache:
```java

//Create NormalizedCacheFactory
NormalizedCacheFactory cacheFactory = new LruNormalizedCacheFactory(EvictionPolicy.builder().maxSizeBytes(10 * 1024).build());

//Create the cache key resolver
CacheKeyResolver<Map<String, Object>> resolver =  new CacheKeyResolver<Map<String, Object>>() {
          @Nonnull @Override public CacheKey resolve(@Nonnull Map<String, Object> objectSource) {
            String id = (String) objectSource.get("id");
            if (id == null || id.isEmpty()) {
              return CacheKey.NO_KEY;
            }
            return CacheKey.from(id);
          }
        }

//Build the Apollo Client
ApolloClient apolloClient = ApolloClient.builder()
                                    .serverUrl("/")
                                    .normalizedCache(cacheFactory, resolver)
                                    .okHttpClient(okHttpClient)
                                    .build();

```

For concrete examples of using response caches, please see the following tests in the [`apollo-integration`](apollo-integration) module:
`CacheTest`, `SqlNormalizedCacheTest`, `LruNormalizedCacheTest`. 

## RxJava Support

Apollo GraphQL client comes with RxJava1 & RxJava2 support. Apollo types such as ApolloCall, ApolloPrefetch & ApolloWatcher can be converted
to their corresponding RxJava1 & RxJava2 Observable types by using wrapper functions provided in RxApollo & Rx2Apollo classes respectively.

### Usage

Converting ApolloCall to a Single:
```java
//Create a query object
EpisodeHeroName query = EpisodeHeroName.builder().episode(Episode.EMPIRE).build();

//Create an ApolloCall object
ApolloCall<EpisodeHeroName.Data> apolloCall = apolloClient.newCall(query);

//RxJava1 Single
Single<EpisodeHeroName.Data> single1 = RxApollo.from(apolloCall);

//RxJava2 Single
Single<EpisodeHeroName.Data> single2 = Rx2Apollo.from(apolloCall);
```

Converting ApolloPrefetch to a Completable:
```java
//Create a query object
EpisodeHeroName query = EpisodeHeroName.builder().episode(Episode.EMPIRE).build();

//Create an ApolloPrefetch object
ApolloPrefetch<EpisodeHeroName.Data> apolloPrefetch = apolloClient.prefetch(query);

//RxJava1 Completable
Completable completable1 = RxApollo.from(apolloPrefetch);

//RxJava2 Completable
Completable completable2 = Rx2Apollo.from(apolloPrefetch);
```

Converting ApolloWatcher to an Observable:
```java
//Create a query object
EpisodeHeroName query = EpisodeHeroName.builder().episode(Episode.EMPIRE).build();

//Create an ApolloWatcher object
ApolloWatcher<EpisodeHeroName.Data> apolloWatcher = apolloClient.newCall(query).watcher();

//RxJava1 Observable
Observable<EpisodeHeroName.Data> observable1 = RxApollo.from(apolloWatcher);

//RxJava2 Observable
Observable<EpisodeHeroName.Data> observable1 = Rx2Apollo.from(apolloWatcher);
```

Also, don't forget to dispose of your Observer/Subscriber when you are finished:
```java
Disposable disposable = Rx2Apollo.from(query).subscribe();

//Dispose of your Observer when you are done with your work
disposable.dispose();
```
As an alternative, multiple Disposables can be collected to dispose of at once via `CompositeDisposable`:
```java
CompositeDisposable disposable = new CompositeDisposable();
disposable.add(Rx2Apollo.from(call).subscribe());

// Dispose of all collected Disposables at once
disposable.clear();
```


For a concrete example of using Rx wrappers for apollo types, checkout the sample app in the [`apollo-sample`](apollo-sample) module.

### Download

RxJava1:

[![Maven Central](https://img.shields.io/maven-central/v/com.apollographql.apollo/apollo-rx-support.svg)](http://repo1.maven.org/maven2/com/apollographql/apollo/apollo-rx-support/)
```gradle
compile 'com.apollographql.apollo:apollo-rx-support:x.y.z'
```

RxJava2:

[![Maven Central](https://img.shields.io/maven-central/v/com.apollographql.apollo/apollo-rx2-support.svg)](http://repo1.maven.org/maven2/com/apollographql/apollo/apollo-rx2-support/)
```gradle
compile 'com.apollographql.apollo:apollo-rx2-support:x.y.z'
```

## License

```
       

Apache License
                           Version 2.0, January 2004
                        https://www.apache.org/licenses/

   TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

   1. Definitions.

      "License" shall mean the terms and conditions for use, reproduction,
      and distribution as defined by Sections 1 through 9 of this document.

      "Licensor" shall mean the copyright owner or entity authorized by
      the copyright owner that is granting the License.

      "Legal Entity" shall mean the union of the acting entity and all
      other entities that control, are controlled by, or are under common
      control with that entity. For the purposes of this definition,
      "control" means (i) the power, direct or indirect, to cause the
      direction or management of such entity, whether by contract or
      otherwise, or (ii) ownership of fifty percent (50%) or more of the
      outstanding shares, or (iii) beneficial ownership of such entity.

      "You" (or "Your") shall mean an individual or Legal Entity
      exercising permissions granted by this License.

      "Source" form shall mean the preferred form for making modifications,
      including but not limited to software source code, documentation
      source, and configuration files.

      "Object" form shall mean any form resulting from mechanical
      transformation or translation of a Source form, including but
      not limited to compiled object code, generated documentation,
      and conversions to other media types.

      "Work" shall mean the work of authorship, whether in Source or
      Object form, made available under the License, as indicated by a
      copyright notice that is included in or attached to the work
      (an example is provided in the Appendix below).

      "Derivative Works" shall mean any work, whether in Source or Object
      form, that is based on (or derived from) the Work and for which the
      editorial revisions, annotations, elaborations, or other modifications
      represent, as a whole, an original work of authorship. For the purposes
      of this License, Derivative Works shall not include works that remain
      separable from, or merely link (or bind by name) to the interfaces of,
      the Work and Derivative Works thereof.

      "Contribution" shall mean any work of authorship, including
      the original version of the Work and any modifications or additions
      to that Work or Derivative Works thereof, that is intentionally
      submitted to Licensor for inclusion in the Work by the copyright owner
      or by an individual or Legal Entity authorized to submit on behalf of
      the copyright owner. For the purposes of this definition, "submitted"
      means any form of electronic, verbal, or written communication sent
      to the Licensor or its representatives, including but not limited to
      communication on electronic mailing lists, source code control systems,
      and issue tracking systems that are managed by, or on behalf of, the
      Licensor for the purpose of discussing and improving the Work, but
      excluding communication that is conspicuously marked or otherwise
      designated in writing by the copyright owner as "Not a Contribution."

      "Contributor" shall mean Licensor and any individual or Legal Entity
      on behalf of whom a Contribution has been received by Licensor and
      subsequently incorporated within the Work.

   2. Grant of Copyright License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      copyright license to reproduce, prepare Derivative Works of,
      publicly display, publicly perform, sublicense, and distribute the
      Work and such Derivative Works in Source or Object form.

   3. Grant of Patent License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      (except as stated in this section) patent license to make, have made,
      use, offer to sell, sell, import, and otherwise transfer the Work,
      where such license applies only to those patent claims licensable
      by such Contributor that are necessarily infringed by their
      Contribution(s) alone or by combination of their Contribution(s)
      with the Work to which such Contribution(s) was submitted. If You
      institute patent litigation against any entity (including a
      cross-claim or counterclaim in a lawsuit) alleging that the Work
      or a Contribution incorporated within the Work constitutes direct
      or contributory patent infringement, then any patent licenses
      granted to You under this License for that Work shall terminate
      as of the date such litigation is filed.

   4. Redistribution. You may reproduce and distribute copies of the
      Work or Derivative Works thereof in any medium, with or without
      modifications, and in Source or Object form, provided that You
      meet the following conditions:

      (a) You must give any other recipients of the Work or
          Derivative Works a copy of this License; and

      (b) You must cause any modified files to carry prominent notices
          stating that You changed the files; and

      (c) You must retain, in the Source form of any Derivative Works
          that You distribute, all copyright, patent, trademark, and
          attribution notices from the Source form of the Work,
          excluding those notices that do not pertain to any part of
          the Derivative Works; and

      (d) If the Work includes a "NOTICE" text file as part of its
          distribution, then any Derivative Works that You distribute must
          include a readable copy of the attribution notices contained
          within such NOTICE file, excluding those notices that do not
          pertain to any part of the Derivative Works, in at least one
          of the following places: within a NOTICE text file distributed
          as part of the Derivative Works; within the Source form or
          documentation, if provided along with the Derivative Works; or,
          within a display generated by the Derivative Works, if and
          wherever such third-party notices normally appear. The contents
          of the NOTICE file are for informational purposes only and
          do not modify the License. You may add Your own attribution
          notices within Derivative Works that You distribute, alongside
          or as an addendum to the NOTICE text from the Work, provided
          that such additional attribution notices cannot be construed
          as modifying the License.

      You may add Your own copyright statement to Your modifications and
      may provide additional or different license terms and conditions
      for use, reproduction, or distribution of Your modifications, or
      for any such Derivative Works as a whole, provided Your use,
      reproduction, and distribution of the Work otherwise complies with
      the conditions stated in this License.

   5. Submission of Contributions. Unless You explicitly state otherwise,
      any Contribution intentionally submitted for inclusion in the Work
      by You to the Licensor shall be under the terms and conditions of
      this License, without any additional terms or conditions.
      Notwithstanding the above, nothing herein shall supersede or modify
      the terms of any separate license agreement you may have executed
      with Licensor regarding such Contributions.

   6. Trademarks. This License does not grant permission to use the trade
      names, trademarks, service marks, or product names of the Licensor,
      except as required for reasonable and customary use in describing the
      origin of the Work and reproducing the content of the NOTICE file.

   7. Disclaimer of Warranty. Unless required by applicable law or
      agreed to in writing, Licensor provides the Work (and each
      Contributor provides its Contributions) on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
      implied, including, without limitation, any warranties or conditions
      of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
      PARTICULAR PURPOSE. You are solely responsible for determining the
      appropriateness of using or redistributing the Work and assume any
      risks associated with Your exercise of permissions under this License.

   8. Limitation of Liability. In no event and under no legal theory,
      whether in tort (including negligence), contract, or otherwise,
      unless required by applicable law (such as deliberate and grossly
      negligent acts) or agreed to in writing, shall any Contributor be
      liable to You for damages, including any direct, indirect, special,
      incidental, or consequential damages of any character arising as a
      result of this License or out of the use or inability to use the
      Work (including but not limited to damages for loss of goodwill,
      work stoppage, computer failure or malfunction, or any and all
      other commercial damages or losses), even if such Contributor
      has been advised of the possibility of such damages.

   9. Accepting Warranty or Additional Liability. While redistributing
      the Work or Derivative Works thereof, You may choose to offer,
      and charge a fee for, acceptance of support, warranty, indemnity,
      or other liability obligations and/or rights consistent with this
      License. However, in accepting such obligations, You may act only
      on Your own behalf and on Your sole responsibility, not on behalf
      of any other Contributor, and only if You agree to indemnify,
      defend, and hold each Contributor harmless for any liability
      incurred by, or claims asserted against, such Contributor by reason
      of your accepting any such warranty or additional liability.

   END OF TERMS AND CONDITIONS

   APPENDIX: How to apply the Apache License to your work.

      To apply the Apache License to your work, attach the following
      boilerplate notice, with the fields enclosed by brackets "[]"
      replaced with your own identifying information. (Don't include
      the brackets!)  The text should be enclosed in the appropriate
      comment syntax for the file format. We also recommend that a
      file or class name and description of purpose be included on the
      same "printed page" as the copyright notice for easier
      identification within third-party archives.

   Copyright [2019] [Rolando Gopez Lacuata]

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       https://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.


```
