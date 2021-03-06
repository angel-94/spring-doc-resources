<p align="center">
    <img alt="Spring Doc Resources" title="Spring Doc Resources" src="https://i.imgur.com/fGi6EaT.png" width="450">
</p>

This project generates and packages the static resources that Spring uses for document production.

## Maven build integration

You can use the [Asciidoctor Maven plugin](https://github.com/asciidoctor/asciidoctor-maven-plugin)
to generate your documentation.

You first need to declare this project as a dependency:

```xml
<dependency>
	<groupId>io.spring.docresources</groupId>
	<artifactId>spring-doc-resources</artifactId>
	<version>${docs.resources.version}</version>
	<type>zip</type>
	<optional>true</optional>
</dependency>
```

Unpack the resources and the actual asciidoc documents in a specific build directory:

```xml
<plugins>
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-dependency-plugin</artifactId>
	<executions>
	<execution>
		<id>unpack-doc-resources</id>
		<goals>
			<goal>unpack-dependencies</goal>
		</goals>
		<phase>generate-resources</phase>
		<configuration>
			<includeGroupIds>io.spring.docresources</includeGroupIds>
			<includeArtifactIds>spring-doc-resources</includeArtifactIds>
			<includeTypes>zip</includeTypes>
			<excludeTransitive>true</excludeTransitive>
			<outputDirectory>${project.build.directory}/refdocs/</outputDirectory>
		</configuration>
	</execution>
</executions>
</plugin>
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-resources-plugin</artifactId>
	<executions>
		<execution>
			<id>copy-asciidoc-resources</id>
			<phase>generate-resources</phase>
			<goals>
				<goal>copy-resources</goal>
			</goals>
			<configuration>
				<outputDirectory>${project.build.directory}/refdocs/</outputDirectory>
				<resources>
					<resource>
						<directory>src/main/asciidoc</directory>
						<filtering>false</filtering>
					</resource>
				</resources>
			</configuration>
		</execution>
	</executions>
</plugin>
</plugins>
```

Finally, launch the documentation generation process; default output location is `target/generated-docs/`:

```xml
<plugin>
	<groupId>org.asciidoctor</groupId>
	<artifactId>asciidoctor-maven-plugin</artifactId>
	<configuration>
		<sourceDirectory>${refdocs.build.directory}</sourceDirectory>
		<attributes>
			// Add attributes generated by the build
			<spring-project-docs-version>${revision}</spring-boot-docs-version>
			<spring-version>${spring.version}</spring-version>
			// Add locations of code snippets to include in the documentation
			<sources-root>${project.basedir}/src/</sources-root>
		</attributes>
	</configuration>
	<executions>
		<execution>
			<id>generate-html-documentation</id>
			<phase>prepare-package</phase>
			<goals>
				<goal>process-asciidoc</goal>
			</goals>
			<configuration>
				<backend>html5</backend>
				<sourceHighlighter>highlight.js</sourceHighlighter>
				<doctype>book</doctype>
				<attributes>
					// these attributes are required to use the doc resources
					<docinfo>shared</docinfo>
					<stylesdir>css/</stylesdir>
					<stylesheet>spring.css</stylesheet>
					<linkcss>true</linkcss>
					<icons>font</icons>
					<highlightjsdir>js/highlight</highlightjsdir>
					<highlightjs-theme>github</highlightjs-theme>
				</attributes>
			</configuration>
		</execution>
	</executions>
</plugin>
```


## Gradle build integration

You can use the [Asciidoctor Gradle plugin](https://asciidoctor.org/docs/asciidoctor-gradle-plugin/)
to generate your documentation.

```groovy
// declare a configuration for documentation resources
configurations {
	docs
}

dependencies {
	docs "io.spring.docresources:spring-doc-resources:${docResourcesVersion}@zip"
}

task prepareAsciidocBuild(type: Sync) {
	dependsOn configurations.docs
	// copy doc resources
	from {
		configurations.docs.collect { zipTree(it) }
	}
	// and doc sources
	from "src/docs/asciidoc/"
	// to a build directory of your choice
	into "$buildDir/asciidoc/build"
}

task('makePDF', type: org.asciidoctor.gradle.AsciidoctorTask){
	dependsOn prepareAsciidocBuild
	backends 'pdf'
	sourceDir "$buildDir/asciidoc/assemble"
	sources {
		include 'index-test.adoc'
		include 'test.adoc'
	}
	options doctype: 'book', eruby: 'erubis'
	logDocuments = true
	attributes 'icons': 'font',
		'sectanchors': '',
		'toc': '',
		'source-highlighter' : 'coderay'
}

asciidoctor {
	// run asciidoctor from that directory
	sourceDir "$buildDir/asciidoc/build"
	sources {
		include '*.adoc'
	}
	resources {
		from(sourceDir) {
			include 'images/*', 'css/**', 'js/**'
		}
	}
	logDocuments = true
	backends = ["html5"]
	options doctype: 'book', eruby: 'erubis'
	attributes  'docinfo': 'shared',
		// use provided stylesheet
		stylesdir: "css/",
		stylesheet: 'spring.css',
		'linkcss': true,
		'icons': 'font',
		// use provided highlighter
		'source-highlighter=highlight.js',
		'highlightjsdir=js/highlight',
		'highlightjs-theme=github'
}

asciidoctor.dependsOn makePDF
```

## Features

spring-doc-resources has a few features that we have added to address certain use cases.

### The "Back to Index" Link

For HTML output, if the current page is not index.html, you automatically get a link to index.html.
This link appears above the table of contents.
For many projects, this link never appears, because that project's build renders the documentation as index.html.

You can customize the destination of the "Back to Index" link by specifying a role with a value of `#index-link`, as follows:

```
[#index-link]
https://spring.io
```

where `https://spring.io` is the link you want.

Please do use a link that readers might reasonably think would be an index page.
(The canonical case is the project's page on spring.io.)

Nominally, you can put that role anywhere, but near the top of your main Asciidoc file makes the most sense.

## Limitations

As with anything, there are some limitations that you should be aware of when you use spring-doc-resources.

### Code samples

When including code samples in the documentation, their location must not rely on relative paths,
as the actual documentation build happens within the `build`/`target` folder.
To work around that limitation, the build should introduce an attribute pointing to a particular
location within project sources, from which code samples can be resolved.

### DocInfo files

To get the dynamic table of contents to work correctly, you need to set the `docinfo` attribute to `shared`, thus: `:docinfo: shared`.
Bear in mind that, if you set the attribute in your build, it overrides the value in your Asciidoc files.
You may still want to set the attribute in your Asciidoc files, though, if you generate files with the `asciidoctor` command for testing.
You can also use `private` docinfo particular to asciidoc documents (see [docinfo documentation](https://asciidoctor.org/docs/user-manual/#naming-docinfo-files)).


## Distribution Zip

The final distribution zip file contains the following:

```
|- docinfo.html, docinfo-footer.html (shared docinfo HTML files)
|- css/** (stylesheets)
|– js
   |- tocbot/** (navigation in table of contents)
   |- highlight/** (code highlighting)
```

You should unzip the whole archive at the top of your Asciidoc hierarchy (typically `src/main/asciidoc` and typically in the same directory as `index.adoc`).

CAUTION: You cannot let Asciidoctor generate its output and then move the files. The files have to be in position when Asciidoctor runs.


## Doc Resources

This project contains the following:

* `/src/main/sass/**`: The stylesheet for the HTML versions of the Spring reference guides, generated from SASS files.
* `/src/main/resources/js/**`: Javascript libraries for the Table of Contents and code hilighting.
* `/src/main/resources/*.html`: The custom [Asciidoctor docinfo files](https://asciidoctor.org/docs/user-manual/#docinfo-file).
* `build.gradle`: The Gradle build file for this project.
* `gulpfile.js`: The Gulp build that compiles sources into static files
* `package.json`: The dependencies for the NPM-based build
* `README.md`: This file.

## Building the project

Running `./gradlew distZip` will build and package the asciidoctor theme in a zip file in `build/distributions`.
You can also publish that artifact to your local maven repository for testing with `./gradlew publishToMavenLocal`.

When working on the project, one can run the following command:

```
$ ./gradlew npm_run_dev
```

This will start a local server on `http://localhost:8080`, watch files under `src/**` and rebuild automatically.
Please consider installing [the Livereload browser extension](http://livereload.com/) for a better experience.

You can also install the Gulp CLI and directly run build commands:

```
$ npm i -g gulp-cli
$ gulp dev
```
