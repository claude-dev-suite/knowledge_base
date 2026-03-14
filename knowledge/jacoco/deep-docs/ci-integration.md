# JaCoCo CI/CD Integration Guide

## GitHub Actions Integration

### Complete Workflow

```yaml
name: Java CI with Coverage

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'

      - name: Run tests with coverage
        run: mvn clean verify

      - name: Generate JaCoCo Badge
        id: jacoco
        uses: cicirello/jacoco-badge-generator@v2
        with:
          jacoco-csv-file: target/site/jacoco/jacoco.csv
          badges-directory: .github/badges
          generate-branches-badge: true
          generate-summary: true

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: target/site/jacoco/jacoco.xml
          flags: unittests
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: Upload coverage to Coveralls
        uses: coverallsapp/github-action@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          path-to-lcov: target/site/jacoco/jacoco.xml
          format: jacoco

      - name: Comment PR with coverage
        if: github.event_name == 'pull_request'
        uses: madrapps/jacoco-report@v1.6
        with:
          paths: target/site/jacoco/jacoco.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          min-coverage-overall: 80
          min-coverage-changed-files: 80

      - name: Upload coverage reports
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: target/site/jacoco/
```

## SonarQube Integration

### Maven Configuration

```xml
<properties>
    <!-- SonarQube -->
    <sonar.host.url>https://sonarcloud.io</sonar.host.url>
    <sonar.organization>your-org</sonar.organization>
    <sonar.projectKey>your-project-key</sonar.projectKey>

    <!-- JaCoCo -->
    <sonar.java.coveragePlugin>jacoco</sonar.java.coveragePlugin>
    <sonar.dynamicAnalysis>reuseReports</sonar.dynamicAnalysis>
    <sonar.jacoco.reportPath>${project.basedir}/target/jacoco.exec</sonar.jacoco.reportPath>
    <sonar.coverage.jacoco.xmlReportPaths>
        ${project.basedir}/target/site/jacoco/jacoco.xml
    </sonar.coverage.jacoco.xmlReportPaths>

    <!-- Exclusions -->
    <sonar.coverage.exclusions>
        **/config/**,
        **/dto/**,
        **/entity/**,
        **/*MapperImpl.java,
        **/Application.java
    </sonar.coverage.exclusions>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.sonarsource.scanner.maven</groupId>
            <artifactId>sonar-maven-plugin</artifactId>
            <version>3.10.0.2594</version>
        </plugin>
    </plugins>
</build>
```

### GitHub Actions with SonarQube

```yaml
- name: Run tests and SonarQube analysis
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: |
    mvn clean verify sonar:sonar \
      -Dsonar.projectKey=${{ secrets.SONAR_PROJECT_KEY }} \
      -Dsonar.organization=${{ secrets.SONAR_ORG }} \
      -Dsonar.host.url=https://sonarcloud.io
```

## Quality Gates

### Coverage Threshold Check

```xml
<execution>
    <id>check-coverage</id>
    <goals>
        <goal>check</goal>
    </goals>
    <configuration>
        <rules>
            <rule>
                <element>BUNDLE</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.80</minimum>
                    </limit>
                    <limit>
                        <counter>BRANCH</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.70</minimum>
                    </limit>
                    <limit>
                        <counter>COMPLEXITY</counter>
                        <value>TOTALCOUNT</value>
                        <maximum>500</maximum>
                    </limit>
                </limits>
            </rule>

            <!-- Package level rules -->
            <rule>
                <element>PACKAGE</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.75</minimum>
                    </limit>
                </limits>
                <excludes>
                    <exclude>com.company.config</exclude>
                    <exclude>com.company.dto</exclude>
                </excludes>
            </rule>

            <!-- Class level rules -->
            <rule>
                <element>CLASS</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.70</minimum>
                    </limit>
                    <limit>
                        <counter>COMPLEXITY</counter>
                        <value>TOTALCOUNT</value>
                        <maximum>50</maximum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</execution>
```

## Multi-Module Projects

### Aggregate Report

```xml
<!-- Parent pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>
                <execution>
                    <id>report-aggregate</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>report-aggregate</goal>
                    </goals>
                    <configuration>
                        <title>Multi-Module Coverage</title>
                        <footer>Generated by JaCoCo</footer>
                        <dataFileIncludes>
                            <dataFileInclude>**/jacoco.exec</dataFileInclude>
                        </dataFileIncludes>
                        <outputDirectory>
                            ${project.reporting.outputDirectory}/jacoco-aggregate
                        </outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Module Configuration

```xml
<!-- module/pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <executions>
                <execution>
                    <id>prepare-agent</id>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## Coverage Diff in PRs

### Script for Coverage Comparison

```bash
#!/bin/bash
# compare-coverage.sh

BASE_COVERAGE=$(grep -o '<counter type="LINE".*covered="[0-9]*"' target/site/jacoco-base/jacoco.xml | grep -o 'covered="[0-9]*"' | cut -d'"' -f2 | awk '{s+=$1} END {print s}')
BASE_TOTAL=$(grep -o '<counter type="LINE".*missed="[0-9]*"' target/site/jacoco-base/jacoco.xml | grep -o 'missed="[0-9]*"' | cut -d'"' -f2 | awk '{s+=$1} END {print s}')
BASE_RATIO=$(echo "scale=4; $BASE_COVERAGE / ($BASE_COVERAGE + $BASE_TOTAL)" | bc)

PR_COVERAGE=$(grep -o '<counter type="LINE".*covered="[0-9]*"' target/site/jacoco/jacoco.xml | grep -o 'covered="[0-9]*"' | cut -d'"' -f2 | awk '{s+=$1} END {print s}')
PR_TOTAL=$(grep -o '<counter type="LINE".*missed="[0-9]*"' target/site/jacoco/jacoco.xml | grep -o 'missed="[0-9]*"' | cut -d'"' -f2 | awk '{s+=$1} END {print s}')
PR_RATIO=$(echo "scale=4; $PR_COVERAGE / ($PR_COVERAGE + $PR_TOTAL)" | bc)

DIFF=$(echo "scale=2; ($PR_RATIO - $BASE_RATIO) * 100" | bc)

echo "Base coverage: $(echo "scale=2; $BASE_RATIO * 100" | bc)%"
echo "PR coverage: $(echo "scale=2; $PR_RATIO * 100" | bc)%"
echo "Difference: $DIFF%"

if (( $(echo "$DIFF < -1" | bc -l) )); then
    echo "❌ Coverage decreased by more than 1%"
    exit 1
else
    echo "✅ Coverage check passed"
    exit 0
fi
```

### GitHub Action for Coverage Diff

```yaml
- name: Checkout base branch
  uses: actions/checkout@v4
  with:
    ref: ${{ github.base_ref }}
    path: base

- name: Build base coverage
  run: |
    cd base
    mvn clean test
    mkdir -p ../target/site/jacoco-base
    cp -r target/site/jacoco/* ../target/site/jacoco-base/

- name: Checkout PR
  uses: actions/checkout@v4

- name: Build PR coverage
  run: mvn clean test

- name: Compare coverage
  run: bash .github/scripts/compare-coverage.sh
```

## Docker Integration

### Dockerfile with Coverage

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app
COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests=false

# Extract coverage report
RUN mkdir -p /coverage && \
    cp -r target/site/jacoco/* /coverage/

FROM eclipse-temurin:17-jre-alpine
COPY --from=build /app/target/*.jar app.jar
COPY --from=build /coverage /coverage

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Docker Compose for Testing

```yaml
version: '3.8'
services:
  app-test:
    build:
      context: .
      target: build
    volumes:
      - ./target/site/jacoco:/app/target/site/jacoco
    command: mvn clean verify
```

## Coverage Badges

### Generate Badge JSON

```bash
#!/bin/bash
# generate-badge.sh

COVERAGE=$(grep -o '<counter type="LINE".*covered="[0-9]*"' target/site/jacoco/jacoco.xml | \
  grep -o 'covered="[0-9]*"' | cut -d'"' -f2 | awk '{s+=$1} END {print s}')
TOTAL=$(grep -o '<counter type="LINE".*missed="[0-9]*"' target/site/jacoco/jacoco.xml | \
  grep -o 'missed="[0-9]*"' | cut -d'"' -f2 | awk '{s+=$1} END {print s}')
PERCENTAGE=$(echo "scale=1; $COVERAGE * 100 / ($COVERAGE + $TOTAL)" | bc)

if (( $(echo "$PERCENTAGE >= 80" | bc -l) )); then
  COLOR="brightgreen"
elif (( $(echo "$PERCENTAGE >= 60" | bc -l) )); then
  COLOR="yellow"
else
  COLOR="red"
fi

cat > .github/badges/coverage.json <<EOF
{
  "schemaVersion": 1,
  "label": "coverage",
  "message": "${PERCENTAGE}%",
  "color": "${COLOR}"
}
EOF
```

### README Badge

```markdown
![Coverage](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/user/repo/main/.github/badges/coverage.json)
```

## Best Practices

1. ✅ Run coverage on every PR
2. ✅ Fail CI if coverage drops significantly
3. ✅ Generate coverage reports as artifacts
4. ✅ Comment PR with coverage details
5. ✅ Use badges to show coverage status
6. ✅ Integrate with SonarQube for quality gates
7. ✅ Exclude generated code and config files
8. ✅ Set realistic coverage thresholds
9. ✅ Monitor coverage trends over time
10. ✅ Don't aim for 100% coverage

## References

- [JaCoCo Maven Plugin](https://www.jacoco.org/jacoco/trunk/doc/maven.html)
- [GitHub Actions](https://docs.github.com/en/actions)
- [SonarQube](https://docs.sonarqube.org/latest/)
- [Codecov](https://docs.codecov.com/)
