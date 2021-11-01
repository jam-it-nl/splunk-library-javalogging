# Changes made to the library
pom.xml
    <groupId>nl.jam.com.splunk.logging</groupId>
    <artifactId>splunk-library-javalogging</artifactId>
    <version>patched.1.11.0</version>

        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>logging-interceptor</artifactId>
            <version>4.9.1</version>
        </dependency>

        <repositories>
                <repository>
                        <id>repsy</id>
                        <url>https://repo.repsy.io/mvn/jam-it/public</url>
                </repository>
                <repository>
                        <id>splunk-artifactory</id>
                        <name>Splunk Releases</name>
                        <url>https://splunk.jfrog.io/splunk/ext-releases-local</url>
                </repository>
        </repositories>

        <distributionManagement>
                <repository>
                        <id>repsy</id>
                        <url>https://repo.repsy.io/mvn/jam-it/public</url>
                </repository>
        </distributionManagement>

HttpEventCollectorSender
        // Fix for https://github.com/splunk/splunk-library-javalogging/issues/148
        private static final String JsonHttpContentType = "application/json; charset=utf-8";

        private void startHttpClient() {
            ...
            this.configureHttpClientForMutualAuthentication(builder);
            httpClient = builder.build();
        }

        private boolean enableLogging = false;
        private boolean configureHttpClientForMutualAuthentication = false;

        public boolean isConfigureHttpClientForMutualAuthentication() {
            return configureHttpClientForMutualAuthentication;
        }

        public void setConfigureHttpClientForMutualAuthentication(boolean configureHttpClientForMutualAuthentication) {
            this.configureHttpClientForMutualAuthentication = configureHttpClientForMutualAuthentication;
        }

        public boolean isEnableLogging() {
            return enableLogging;
        }

        public void setEnableLogging(boolean enableLogging) {
            this.enableLogging = enableLogging;
        }

        private void configureHttpClientForMutualAuthentication(OkHttpClient.Builder builder) {
        try {
            if (!this.configureHttpClientForMutualAuthentication) {
                Logger.getLogger(this.getClass().getSimpleName()).log(Level.WARNING, "Not configuring Http Client for MutualAuthentication");
                return;
            }

            Logger.getLogger(this.getClass().getSimpleName()).log(Level.WARNING, "Configuring Http Client for MutualAuthentication");

            TrustManager[] trustManagers = this.getTrustManagers();
            SSLContext sslContext = this.getSslContext(trustManagers);

            if (this.enableLogging) {
                HttpLoggingInterceptor httpLoggingInterceptor = new HttpLoggingInterceptor();
                builder.addInterceptor(httpLoggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY));
            }

            builder.sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustManagers[0]);
        } catch (Exception e) {
            throw new RuntimeException(e.getMessage(), e);
        }
    }

    private TrustManager[] getTrustManagers() throws Exception {
        try {
            String trustStore = System.getProperty("javax.net.ssl.trustStore");
            Path keyStorePath = Paths.get(trustStore);

            KeyStore keyStore = KeyStore.getInstance("JKS");
            try (InputStream stream = Files.newInputStream(keyStorePath)) {
                keyStore.load(stream, System.getProperty("javax.net.ssl.trustStorePassword").toCharArray());
            }

            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("SunX509");
            trustManagerFactory.init(keyStore);
            return trustManagerFactory.getTrustManagers();
        } catch (Exception e) {
            Logger.getLogger(this.getClass().getSimpleName()).log(Level.WARNING, "Failed getting TrustManagers, falling back to default: " + e.getMessage());
            return new TrustManager[] {new X509TrustManager() {
                @Override
                public void checkClientTrusted(java.security.cert.X509Certificate[] chain, String authType) throws CertificateException {}

                @Override
                public void checkServerTrusted(java.security.cert.X509Certificate[] chain, String authType) throws CertificateException {}

                @Override
                public java.security.cert.X509Certificate[] getAcceptedIssuers() {
                    return new java.security.cert.X509Certificate[] {};
                }
            }};
        }
    }

    private SSLContext getSslContext(TrustManager[] trustManagers) throws Exception {
        try {
            Path keyStorePath = Paths.get(System.getProperty("javax.net.ssl.keyStore"));
            String clientCertPassword = System.getProperty("javax.net.ssl.keyStorePassword");

            KeyStore keyStore = KeyStore.getInstance("PKCS12");
            try (InputStream stream = Files.newInputStream(keyStorePath)) {
                keyStore.load(stream, clientCertPassword.toCharArray());
            }

            KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance("SunX509");
            keyManagerFactory.init(keyStore, clientCertPassword.toCharArray());

            SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
            sslContext.init(keyManagerFactory.getKeyManagers(), trustManagers, new SecureRandom());

            return sslContext;
        } catch (Exception e) {
            Logger.getLogger(this.getClass().getSimpleName()).log(Level.WARNING, "Failed getting SslContext, falling back to default: " + e.getMessage());
            SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
            sslContext.init(null, trustManagers, new java.security.SecureRandom());
            return sslContext;
        }
    }

# Reference
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
mvn clean install
mvn clean deploy