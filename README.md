# Venafi CodeSign Protect: Gitlab integration

This product integrates [Venafi CodeSign Protect](https://www.venafi.com/platform/code-signing) with Gitlab-based CI/CD processes.

Venafi CodeSign Protect is a solution for securing machines against attacks and exploits, by signing executables, libraries and other machine runtime artifacts with digital signatures. Unlike naive methods of code signing, Venafi CodeSign Protect is more secure, by storing and securing the signing key separately from the CI/CD infrastructure (perhaps even in a Hardware Security Module) and by providing access control to signing keys. It also provides important insights to security teams, such as how and when signing keys are used.

This product allows one to sign and verify files through Venafi CodeSign Protect. The following signing tools are currently supported:

 * Jarsigner (Java)
 * Signtool (Windows)

**Table of contents**

 - [Usage overview](#usage-overview)
 - [Setting up executor hosts (shell and SSH executors only)](#setting-up-executor-hosts-shell-and-ssh-executors-only)
 - [Compatibility](#compatibility)
 - [Usage](#usage)
   - [Sign with Jarsigner](#sign-with-jarsigner)
     - [Docker executor](#docker-executor)
     - [Shell or SSH executor](#shell-or-ssh-executor)
     - [Variables](#variables)
   - [Verify with Jarsigner](#verify-with-jarsigner)
     - [Docker executor](#docker-executor-1)
     - [Shell or SSH executor](#shell-or-ssh-executor-1)
     - [Variables](#variables-1)
   - [Sign with Signtool](#sign-with-signtool)
     - [Docker executor](#docker-executor-2)
     - [Shell or SSH executor](#shell-or-ssh-executor-2)
     - [Variables](#variables-2)
   - [Verify with Signtool](#verify-with-signtool)
     - [Docker executor](#docker-executor-3)
     - [Shell or SSH executor](#shell-or-ssh-executor-3)
     - [Variables](#variables-3)
 - [Signtool caveats](#signtool-caveats)
 - [Contribution & development](#contribution-development)

## Usage overview

You must already have access to one or more Venafi Trust Protection Platforms™ (TPPs). This Gitlab integration product requires you to specify TPP address and authentication details.

You use this Gitlab integration product by defining, inside your Gitlab CI YAML, steps that perform signing or verification. These steps consume artifacts generated by previous steps (such as unsigned .jar files), and may output artifacts that you can use in later steps (such as signed .jar files).

## Setting up executor hosts (shell and SSH executors only)

If you plan on using this Gitlab integration product in combination with the shell and SSH executors, then you must install the following software on the hosts on which those executors operate. This Gitlab integration product does not take care of installing these prerequisites for you.

 * Install Venafi CodeSign Protect client tools (see [Compatibility](#compatibility) to learn which versions are supported)
    - You do *not* need to *configure* the client tools (i.e. they don't need to be configured with a TPP address or credentials). They just need to be installed. This Gitlab integration product will take care of configuring the client tools with specific TPPs.
 * Install one or more signing tools, e.g. [Jarsigner](https://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html) (part of the JDK) or [Signtool](https://docs.microsoft.com/en-us/dotnet/framework/tools/signtool-exe) (part of the Windows 10 SDK).
 * Install Python >= 3.8. Ensure that it's in PATH.
 * Install our Gitlab integration package: `pip install venafi-codesigning-gitlab-integration`

## Compatibility

This product is compatible with:

 * Trust Protection Platform 20.2 or later.
 * Venafi CodeSign Protect client tools 20.2 or later.

This product supports the following Gitlab runner executors:

 * Shell
 * SSH
 * Docker

## Usage

### Sign with Jarsigner

This section shows how to sign one or more files with Java's [Jarsigner](https://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html) tool. It assumes that jarsigner is in PATH.

#### Docker executor

 * Define a job that calls `venafi-sign-jarsigner`.
 * Ensure the job operates within the image `quay.io/fullstaq-venafi-codesigning-gitlab-integration/jarsigner-x86_64`.
 * Set the `INPUT_PATH` or `INPUT_GLOB` variable to the file(s) that you wish to sign.
 * Set other required variables too. See the variables reference below.

~~~yaml
stages:
  - build
  - sign

# Build a 'foo.jar' and pass it as an artifact to the 'sign' stage.
build_jar:
  stage: build
  script:
    - echo 'public class Foo { public static void main() { } }' > Foo.java
    - javac Foo.java
    - jar -cf foo.jar Foo.class
  artifacts:
    paths:
      - foo.jar

# Sign the 'foo.jar' that was generated by the 'build' stage,
# then store the signed jar as an artifact.
sign_jarsigner:
  stage: sign
  image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/jarsigner-x86_64
  script:
    - venafi-sign-jarsigner
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT_PATH: foo.jar
    CERTIFICATE_LABEL: my label
  artifacts:
    paths:
      - foo.jar
~~~

#### Shell or SSH executor

 * Define a job that calls `venafi-sign-jarsigner`.
 * Set the `INPUT_PATH` or `INPUT_GLOB` variable to the file(s) that you wish to sign.
 * Set other required variables too. See the variables reference below.

~~~yaml
stages:
  - build
  - sign

# Build a 'foo.jar' and pass it as an artifact to the 'sign' stage.
build_jar:
  stage: build
  script:
    - echo 'public class Foo { public static void main() { } }' > Foo.java
    - javac Foo.java
    - jar -cf foo.jar Foo.class
  artifacts:
    paths:
      - foo.jar

# Sign the 'foo.jar' that was generated by the 'build' stage,
# then store the signed jar as an artifact.
sign_jarsigner:
  stage: sign
  script:
    - venafi-sign-jarsigner
  variables:
    TPP_AUTH_URL: https://my-tpp/auth
    TPP_HSM_URL: https://my-tpp/hsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT_PATH: foo.jar
    CERTIFICATE_LABEL: my label
  artifacts:
    paths:
      - foo.jar
~~~

#### Variables

Required:

 * `TPP_AUTH_URL`: The TPP's authorization URL.
 * `TPP_HSM_URL`: The TPP's Hardware Security Module (HSM) backend URL.
 * `TPP_USERNAME`: A login username for the TPP.
 * `TPP_PASSWORD` or `TPP_PASSWORD_BASE64`: The password associated with the login username. You can specify it normally, or in Base64 format. The latter is useful for storing the password in a Gitlab variable, in masked form, because Gitlab can only mask variables whose content only consists of Base64 characters.
 * `INPUT_PATH` or `INPUT_GLOB`: Specifies the file(s) to sign, either through a single filename, or a glob.
 * `CERTIFICATE_LABEL`: The label of the certificate (inside the TPP) to use for code signing. You can obtain a list of labels with `pkcs11config listcertificates`.

Optional:

 * `TIMESTAMPING_SERVERS`: Specifies one or more timestamping authority servers to use during signing. Specifying this is strongly recommended, because it allows signed files to be usable even after the original signing certificate has expired.

    If you specify more than one server, then a random one will be used.

    Example:

    ~~~
    TIMESTAMPING_SERVERS: http://server1,http://server2
    ~~~

    **Tip:** here are some public timestamping authorities that you can use:

     - http://timestamp.digicert.com
     - http://timestamp.globalsign.com
     - http://timestamp.comodoca.com/authenticode
     - http://tsa.starfieldtech.com

 * `EXTRA_ARGS`: Specifies extra custom CLI arguments to pass to Jarsigner. The arguments are comma-separated.

    These arguments will be _appended_ to the Jarsigner CLI invocation, and take precedence over any arguments implicitly passed by this plugin.

    Example:

    ~~~
    EXTRA_ARGS: -arg1,-arg2
    ~~~

 * `EXTRA_TRUSTED_TLS_CA_CERTS` (only applicable when using Docker): Allows registering extra TLS CA certificates into the truststore. This is useful if your TPP's TLS certificate is not recognized by the TLS trust store in our Docker image.

   Set the value to the path of a .pem file that contains one or more certificates to add to the trust store.

   The certificates are added to the truststore during execution of `venafi-sign-jarsigner`. So the recommended way to use this feature, is by adding an additional command — prior to the execution of `venafi-sign-jarsigner` — to fetch the CA certificate file and to place it at the expected location. Example:

   ~~~yaml
   sign_jarsigner:
     image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/jarsigner-x86_64
     script:
       - wget -O /downloaded-ca.crt https://internal.company.url/path-to-your-ca-chain.crt
       - venafi-sign-jarsigner
     variables:
       TPP_AUTH_URL: https://my-tpp/vedauth
       TPP_HSM_URL: https://my-tpp/vedhsm
       TPP_USERNAME: my_username
       # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

       INPUT_PATH: foo.jar
       CERTIFICATE_LABEL: my label
       EXTRA_TRUSTED_TLS_CA_CERTS: /downloaded-ca.crt
   ~~~

 * `VENAFI_CLIENT_TOOLS_DIR`: Specifies the path to the directory in which Venafi CodeSign Protect client tools are installed. If not specified, it's autodetected as follows:

     - Linux: /opt/venafi/codesign
     - macOS: /Library/Venafi/CodeSigning
     - Windows: autodetected from the registry, or (if that fails): C:\Program Files\Venafi CodeSign Protect

### Verify with Jarsigner

This section shows how to verify one or more files with Java's [Jarsigner](https://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html) tool. It assumes that jarsigner is in PATH.

#### Docker executor

 * Define a job that calls `venafi-verify-jarsigner`.
 * Ensure the job operates within the image `quay.io/fullstaq-venafi-codesigning-gitlab-integration/jarsigner-x86_64`.
 * Set the `INPUT_PATH` or `INPUT_GLOB` variable to the file(s) that you wish to verify.
 * Set other required variables too. See the variables reference below.

~~~yaml
stages:
  - fetch
  - verify

# Fetch a signed 'signed.jar' and pass it as an artifact to the 'verify' stage.
fetch_jar:
  stage: fetch
  script:
    - wget https://internal.lan/signed.jar
  artifacts:
    paths:
      - signed.jar

# Verify 'signed.jar' that was fetched by the 'fetch' stage.
verify_jarsigner:
  stage: verify
  image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/jarsigner-x86_64
  script:
    - venafi-verify-jarsigner
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT_PATH: signed.jar
    CERTIFICATE_LABEL: my label
~~~

#### Shell or SSH executor

 * Define a job that calls `venafi-verify-jarsigner`.
 * Set the `INPUT_PATH` or `INPUT_GLOB` variable to the file(s) that you wish to verify.
 * Set other required variables too. See the variables reference below.
 * *Note*: the host which performs the verification does not need to have pre-installed the certificate against which to verify. We'll will fetch the certificate from the TPP, which is why it requires a certificate label.

~~~yaml
stages:
  - fetch
  - verify

# Fetch a signed 'signed.jar' and pass it as an artifact to the 'verify' stage.
fetch_jar:
  stage: fetch
  script:
    - wget https://internal.lan/signed.jar
  artifacts:
    paths:
      - signed.jar

# Verify 'signed.jar' that was fetched by the 'fetch' stage.
verify_jarsigner:
  stage: verify
  script:
    - venafi-verify-jarsigner
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT_PATH: signed.jar
    CERTIFICATE_LABEL: my label
~~~

#### Variables

Required:

 * `TPP_AUTH_URL`: The TPP's authorization URL.
 * `TPP_HSM_URL`: The TPP's Hardware Security Module (HSM) backend URL.
 * `TPP_USERNAME`: A login username for the TPP.
 * `TPP_PASSWORD` or `TPP_PASSWORD_BASE64`: The password associated with the login username. You can specify it normally, or in Base64 format. The latter is useful for storing the password in a Gitlab variable, in masked form, because Gitlab can only mask variables whose content only consists of Base64 characters.
 * `INPUT_PATH` or `INPUT_GLOB`: Specifies the file(s) to verify, either through a single filename, or a glob.
 * `CERTIFICATE_LABEL`: The label of the certificate (inside the TPP) that was used for signing the file(s). You can obtain a list of labels with `pkcs11config listcertificates`.

Optional:

 * `EXTRA_TRUSTED_TLS_CA_CERTS` (only applicable when using Docker): Allows registering extra TLS CA certificates into the truststore. This is useful if your TPP's TLS certificate is not recognized by the TLS trust store in our Docker image.

   Set the value to the path of a .pem file that contains one or more certificates to add to the trust store.

   The certificates are added to the truststore during execution of `venafi-verify-jarsigner`. So the recommended way to use this feature, is by adding an additional command — prior to the execution of `venafi-verify-jarsigner` — to fetch the CA certificate file and to place it at the expected location. Example:

   ~~~yaml
   verify_jarsigner:
     image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/jarsigner-x86_64
     script:
       - wget -O /downloaded-ca.crt https://internal.company.url/path-to-your-ca-chain.crt
       - venafi-verify-jarsigner
     variables:
       TPP_AUTH_URL: https://my-tpp/vedauth
       TPP_HSM_URL: https://my-tpp/vedhsm
       TPP_USERNAME: my_username
       # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

       INPUT_PATH: signed.jar
       CERTIFICATE_LABEL: my label
       EXTRA_TRUSTED_TLS_CA_CERTS: /downloaded-ca.crt
   ~~~

 * `VENAFI_CLIENT_TOOLS_DIR`: Specifies the path to the directory in which Venafi CodeSign Protect client tools are installed. If not specified, it's autodetected as follows:

     - Linux: /opt/venafi/codesign
     - macOS: /Library/Venafi/CodeSigning
     - Windows: autodetected from the registry, or (if that fails): C:\Program Files\Venafi CodeSign Protect

### Sign with Signtool

This section shows how to sign one or more files with Microsoft's [Signtool](https://docs.microsoft.com/en-us/dotnet/framework/tools/signtool-exe) tool.

#### Docker executor

Usage instructions:

 * Define a job that calls `venafi-sign-signtool`.
 * Ensure that this job runs on a Windows-based runner, by setting the proper tags.
 * Ensure this job operates within the image `quay.io/fullstaq-venafi-codesigning-gitlab-integration/signtool-x86_64`.
 * Set the `INPUT` variable to a filename or a glob that you wish to sign.
 * Set other required variables too. See the variables reference below.

*Notes*:

 - Our image uses x64 signtool by default. If you wish to use a different architecture's signtool, then set `SIGNTOOL_PATH` to `C:\winsdk\<arch>\signtool`.
 - We use 'sha256' as the default signature digest algorithm, unlike Signtool's default ('sha1'). You may want to override this if you care about compatibility with older Windows versions that didn't support SHA-256.
 - Please read the [Signtool caveats](#signtool-caveats).

~~~yaml
stages:
  - build
  - sign

# Build a 'foo.exe' and pass it as an artifact to the 'sign' stage.
build_exe:
  stage: build
  image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/signtool-x86_64
  tags:
    - windows
  script:
    - copy C:\Windows\System32\Notepad.exe foo.exe
  artifacts:
    paths:
      - foo.exe

# Sign the 'foo.exe' that was generated by the 'build' stage,
# then store the signed exe as an artifact.
sign_signtool:
  stage: sign
  image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/signtool-x86_64
  tags:
    - windows
  script:
    - venafi-sign-signtool
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT: foo.exe
    CERTIFICATE_SUBJECT_NAME: mydomain.com
  artifacts:
    paths:
      - foo.exe
~~~

#### Shell or SSH executor

Usage instructions:

 * Define a job that calls `venafi-sign-signtool`.
 * Ensure that this job runs on a Windows-based runner, by setting the proper tags.
 * Set the `INPUT` variable to a filename or a glob that you wish to sign.
 * Set other required variables too. See the variables reference below.

*Notes*:

 - We assume that signtool.exe is in PATH, unless you explicitly specify its path with `SIGNTOOL_PATH`.
 - We use 'sha256' as the default signature digest algorithm, unlike Signtool's default ('sha1'). You may want to override this if you care about compatibility with older Windows versions that didn't support SHA-256.
 - Please read the [Signtool caveats](#signtool-caveats).

~~~yaml
stages:
  - build
  - sign

# Build a 'foo.exe' and pass it as an artifact to the 'sign' stage.
build_exe:
  stage: build
  tags:
    - windows
  script:
    - copy C:\Windows\System32\Notepad.exe foo.exe
  artifacts:
    paths:
      - foo.exe

# Sign the 'foo.exe' that was generated by the 'build' stage,
# then store the signed exe as an artifact.
sign_signtool:
  stage: sign
  tags:
    - windows
  script:
    - venafi-sign-signtool
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT: foo.exe
    CERTIFICATE_SUBJECT_NAME: mydomain.com
  artifacts:
    paths:
      - foo.exe
~~~

#### Variables

Required:

 * `TPP_AUTH_URL`: The TPP's authorization URL.
 * `TPP_HSM_URL`: The TPP's Hardware Security Module (HSM) backend URL.
 * `TPP_USERNAME`: A login username for the TPP.
 * `TPP_PASSWORD` or `TPP_PASSWORD_BASE64`: The password associated with the login username. You can specify it normally, or in Base64 format. The latter is useful for storing the password in a Gitlab variable, in masked form, because Gitlab can only mask variables whose content only consists of Base64 characters.
 * `INPUT`: A path or a glob that specifies the file(s) to be signed.
 * `CERTIFICATE_SUBJECT_NAME` or `CERTIFICATE_SHA1`: Specifies the certificate (inside the TPP) to use for signing.

   You can either specify the certificate's Common Name ("Issued to" or "CN"), or its SHA-1 hash.

   You can obtain a list of Common Names with `cspconfig listcertificates` and checking what comes after `CN=`.

   Specifying the SHA-1 hash is useful if there are multiple certificates with the same Common Name.

Optional:

 * `TIMESTAMPING_SERVERS`: Specifies one or more timestamping authority servers to use during signing. Specifying this is strongly recommended, because it allows signed files to be usable even after the original signing certificate has expired.

    If you specify more than one server, then a random one will be used.

    Example:

    ~~~
    TIMESTAMPING_SERVERS: http://server1,http://server2
    ~~~

    **Tip:** here are some public timestamping authorities that you can use:

     - http://timestamp.digicert.com
     - http://timestamp.globalsign.com
     - http://timestamp.comodoca.com/authenticode
     - http://tsa.starfieldtech.com

 * `SIGNATURE_DIGEST_ALGOS`: The digest algorithm(s) to use to creating signatures.

    If none specified, 'sha256' is used as the default algorithm. This is very secure, but may not be compatible with older Windows versions. If you need compatibility with older Windows versions, you should specify 'sha1,sha256' (in that order).

    When multiple digest algorithms are specified, they are applied in the order specified.

    Example:

    ~~~
    SIGNATURE_DIGEST_ALGOS=sha1,sha256
    ~~~

 * `APPEND_SIGNATURES` (boolean): If the target file(s) already have signatures, then append a new signature instead of overwriting the existing signatures.

   Defaults to `false`.

 * `EXTRA_ARGS`: Specifies extra custom CLI arguments to pass to Signtool. The arguments are comma-separated.

    These arguments will be _appended_ to the Signtool CLI invocation. If they overlap with any arguments implicitly passed by this plugin,
    then Signtool will raise an error.

    Example:

    ~~~
    EXTRA_ARGS: /arg1,/arg2
    ~~~

 * `EXTRA_TRUSTED_TLS_CA_CERTS` (only applicable when using Docker): Allows registering extra TLS CA certificates into the truststore. This is useful if your TPP's TLS certificate is not recognized by the TLS trust store in our Docker image.

   Set the value to the path of a PEM or DER file that contains one or more certificates to add to the trust store.

   The certificates are added to the truststore during execution of `venafi-sign-signtool`. So the recommended way to use this feature, is by adding an additional command — prior to the execution of `venafi-sign-signtool` — to fetch the CA certificate file and to place it at the expected location. Example:

   ~~~yaml
   sign_signtool:
     script:
       - powershell -Command "Invoke-WebRequest -Uri 'https://internal.company.url/path-to-your-ca-chain.crt' -OutFile 'C:\downloaded-ca.crt'"
       - venafi-sign-signtool
     variables:
       TPP_AUTH_URL: https://my-tpp/vedauth
       TPP_HSM_URL: https://my-tpp/vedhsm
       TPP_USERNAME: my_username
       # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

       INPUT: foo.exe
       CERTIFICATE_SUBJECT_NAME: mydomain.com
       EXTRA_TRUSTED_TLS_CA_CERTS: C:\downloaded-ca.crt
   ~~~

 * `SIGNTOOL_PATH`: The full path to signtool.exe. If not specified, we assume that it's in PATH.

 * `VENAFI_CLIENT_TOOLS_DIR`: Specifies the path to the directory in which Venafi CodeSign Protect client tools are installed. If not specified, it's autodetected from the registry. If that fails, we fallback to C:\Program Files\Venafi CodeSign Protect.

 * `MACHINE_CONFIGURATION` (boolean): Whether to load CSP configuration from the machine registry hive instead of the user registry hive. Defaults to false.

### Verify with Signtool

This section shows how to verify one or more files with Microsoft's [Signtool](https://docs.microsoft.com/en-us/dotnet/framework/tools/signtool-exe) tool.

#### Docker executor

Usage instructions:

 * Define a job that calls `venafi-verify-signtool`.
 * Ensure that this job runs on a Windows-based runner, by setting the proper tags.
 * Ensure this job operates within the image `quay.io/fullstaq-venafi-codesigning-gitlab-integration/signtool-x86_64`.
 * Set the `INPUT` variable to a filename or a glob that you wish to verify.
 * Set other required variables too. See the variables reference below.

*Notes*:

 - Our image uses x64 signtool by default. If you wish to use a different architecture's signtool, then set `SIGNTOOL_PATH` to `C:\winsdk\<arch>\signtool`.
 - Please read the [Signtool caveats](#signtool-caveats).

~~~yaml
stages:
  - fetch
  - verify

# Fetch a signed 'signed.exe' and pass it as an artifact to the 'verify' stage.
fetch_exe:
  stage: fetch
  image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/signtool-x86_64
  tags:
    - windows
  script:
    - powershell -Command "Invoke-WebRequest -Uri 'https://internal.lan/signed.exe' -OutFile 'C:\signed.exe'"
  artifacts:
    paths:
      - C:\signed.exe

# Verify 'signed.exe' that was fetched by the 'fetch' stage.
verify_signtool:
  stage: sign
  image: quay.io/fullstaq-venafi-codesigning-gitlab-integration/signtool-x86_64
  tags:
    - windows
  script:
    - venafi-verify-signtool
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT: signed.exe
~~~

#### Shell or SSH executor

Usage instructions:

 * Define a job that calls `venafi-verify-signtool`.
 * Ensure that this job runs on a Windows-based runner, by setting the proper tags.
 * Set the `INPUT` variable to a filename or a glob that you wish to verify.
 * Set other required variables too. See the variables reference below.

*Notes*:

 - We assume that signtool.exe is in PATH, unless you explicitly specify its path with `SIGNTOOL_PATH`.
 - Please read the [Signtool caveats](#signtool-caveats).

~~~yaml
stages:
  - fetch
  - verify

# Fetch a signed 'signed.exe' and pass it as an artifact to the 'verify' stage.
fetch_exe:
  stage: fetch
  tags:
    - windows
  script:
    - powershell -Command "Invoke-WebRequest -Uri 'https://internal.lan/signed.exe' -OutFile 'C:\signed.exe'"
  artifacts:
    paths:
      - C:\signed.exe

# Verify 'signed.exe' that was fetched by the 'fetch' stage.
verify_signtool:
  stage: sign
  tags:
    - windows
  script:
    - venafi-verify-signtool
  variables:
    TPP_AUTH_URL: https://my-tpp/vedauth
    TPP_HSM_URL: https://my-tpp/vedhsm
    TPP_USERNAME: my_username
    # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

    INPUT: signed.exe
~~~

#### Variables

Required:

 * `TPP_AUTH_URL`: The TPP's authorization URL.
 * `TPP_HSM_URL`: The TPP's Hardware Security Module (HSM) backend URL.
 * `TPP_USERNAME`: A login username for the TPP.
 * `TPP_PASSWORD` or `TPP_PASSWORD_BASE64`: The password associated with the login username. You can specify it normally, or in Base64 format. The latter is useful for storing the password in a Gitlab variable, in masked form, because Gitlab can only mask variables whose content only consists of Base64 characters.
 * `INPUT`: A path or a glob that specifies the file(s) to verify.

Optional:

 * `EXTRA_TRUSTED_TLS_CA_CERTS` (only applicable when using Docker): Allows registering extra TLS CA certificates into the truststore. This is useful if your TPP's TLS certificate is not recognized by the TLS trust store in our Docker image.

   Set the value to the path of a PEM or DER file that contains one or more certificates to add to the trust store.

   The certificates are added to the truststore during execution of `venafi-verify-signtool`. So the recommended way to use this feature, is by adding an additional command — prior to the execution of `venafi-verify-signtool` — to fetch the CA certificate file and to place it at the expected location. Example:

   ~~~yaml
   verify_signtool:
     script:
       - powershell -Command "Invoke-WebRequest -Uri 'https://internal.company.url/path-to-your-ca-chain.crt' -OutFile 'C:\downloaded-ca.crt'"
       - venafi-verify-signtool
     variables:
       TPP_AUTH_URL: https://my-tpp/vedauth
       TPP_HSM_URL: https://my-tpp/vedhsm
       TPP_USERNAME: my_username
       # TPP_PASSWORD or TPP_PASSWORD_BASE64 should be set in the UI, with masking enabled.

       INPUT: signed.exe
       EXTRA_TRUSTED_TLS_CA_CERTS: C:\downloaded-ca.crt
   ~~~

 * `SIGNTOOL_PATH`: The full path to signtool.exe. If not specified, we assume that it's in PATH.

 * `VENAFI_CLIENT_TOOLS_DIR`: Specifies the path to the directory in which Venafi CodeSign Protect client tools are installed. If not specified, it's autodetected from the registry. If that fails, we fallback to C:\Program Files\Venafi CodeSign Protect.

## Signtool caveats

When using Signtool, you must ensure that all your TPP environments disable the option "Include Certificate Chain". You must do this for *all* TPP environments that the authenticated user account has access to: not just the ones to be used together with Signtool.

If you do not do this, then Signtool will trigger a confirmation dialog box, in which Windows asks for approval to import root certificates. Signtool can't continue until a human manually clicks "Yes".

This is especially problematic when using Signtool in a container, because there is no user interface, so it's impossible to click on anything.

## Contribution & development

See the [contribution guide](CONTRIBUTING.md).
