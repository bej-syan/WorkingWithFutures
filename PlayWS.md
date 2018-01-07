# Play WS

**Play WS** is a powerful HTTP client library, originally developed by the Play team for use with Play Framework.

It uses `AsyncHttpClient` for HTTP client functionality and has no Play dependencies. 

#### HOW TO USE

    // Add play-ahc-ws-standalone as a dependency in SBT:
    libraryDependencies += "com.typesafe.play" %% "play-ahc-ws-standalone" % "LASTEST_VERSION"

This adds the standalone version of Play WS, backed by AsyncHttpClient. 

##### To add XML and JSON support using Play-JSON or Scala XML, add the following:

    libraryDependencies += "com.typesafe.play" %% "play-ws-standalone-xml" % playWsStandaloneVersion
    libraryDependencies += "com.typesafe.play" %% "play-ws-standalone-json" % playWsStandaloneVersion

