# Final Project
## Prerequisites

### Operating System

**Ubuntu 22.04**

### Docker & Docker Compose

- **Docker & Docker Compose**: Required to orchestrate and deploy the 5G Core and UERANSIM containers.
    * **Installation Guide**: [Official Docker Install for Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
    

### Python

- **Python 3.8+**: Ensure you have Python 3 installed on your machine.
- **Required Libraries**: The script relies on specific packages for data manipulation and plotting. Install them via `pip`:
    
    ```bash
    pip install pandas numpy matplotlib
    ```
    

### gtp5g
    ```bash
    git clone --branch v0.9.16 https://github.com/free5gc/gtp5g.git
    ```
    

## Getting Started

- Build/Get Core Network images
    - free5GC
    
    ```bash
    cd free5gc
    cd base
    git clone --recursive -j `nproc` https://github.com/free5gc/free5gc.git
    cd ..
    make
    docker compose -f docker-compose-build.yaml build
    ```
- Factor 1
    - modify free5gc/base/free5gc/NFs/amf/internal/ngap/service/service.go
    ```go
    func handleConnection(conn *sctp.SCTPConn, bufsize uint32, handler NGAPHandler) {
        defer func() {
            if p := recover(); p != nil {
                // Print stack for panic to log. Fatalf() will let program exit.
                logger.NgapLog.Fatalf("panic: %v\n%s", p, string(debug.Stack()))
            }

            // if AMF call Stop(), then conn.Close() will return EBADF because conn has been closed inside Stop()
            if err := conn.Close(); err != nil && !errors.Is(err, syscall.EBADF) {
                logger.NgapLog.Errorf("close connection error: %+v", err)
            }
            connections.Delete(conn)
        }()

        for {
            buf := make([]byte, bufsize)

            n, info, notification, err := conn.SCTPRead(buf)
            if err != nil {
                switch classifySCTPReadError(err) {
                case sctpReadEOF:
                    logger.NgapLog.Debugln("Read EOF from client")
                    handler.HandleConnectionError(conn)
                    return
                case sctpReadTimeout:
                    logger.NgapLog.Debugln("SCTP read timeout")
                    continue
                case sctpReadInterrupted:
                    logger.NgapLog.Debugf("SCTPRead: %+v", err)
                    continue
                case sctpReadClosed:
                    logger.NgapLog.Debugln("SCTP connection already closed")
                    return
                case sctpReadFailed:
                    logger.NgapLog.Errorf(
                        "Handle connection[addr: %+v] error: %+v",
                        conn.RemoteAddr(),
                        err,
                    )
                    handler.HandleConnectionError(conn)
                    return
                }
            }

            if notification != nil {
                if handler.HandleNotification != nil {
                    handler.HandleNotification(conn, notification)
                } else {
                    logger.NgapLog.Warnf("Received sctp notification[type 0x%x] but not handled", notification.Type())
                }
            } else {
                if info == nil || info.PPID != ngapMessage.PPID {
                    logger.NgapLog.Warnln("Received SCTP PPID != 60, discard this packet")
                    continue
                }

                logger.NgapLog.Tracef("Read %d bytes", n)
                logger.NgapLog.Tracef("Packet content:\n%+v", hex.Dump(buf[:n]))

                // Dispatch message through worker pool for parallel processing
                // dispatchToWorkerPool(conn, buf[:n], handler)
                handler.HandleMessage(conn, buff[:n])
            }
        }
    }
    ```
    - build image
    ```bash
    cd free5gc
    make amf
    docker compose -f docker-compose-build.yaml build
    ```
- Factor 2
    - modify free5gc/base/free5gc/NFs/amf/internal/ngap/service/service.go
    ```go
    func listenAndServe(addr *sctp.SCTPAddr, handler NGAPHandler, sctpConfig *sctp.SocketConfig) {
        defer func() {
            if p := recover(); p != nil {
                // Print stack for panic to log. Fatalf() will let program exit.
                logger.NgapLog.Fatalf("panic: %v\n%s", p, string(debug.Stack()))
            }
        }()

        if sctpConfig == nil {
            logger.NgapLog.Errorf("Error sctp SocketConfig is nil")
            return
        }

        if listener, err := sctpConfig.Listen("sctp", addr); err != nil {
            logger.NgapLog.Errorf("Failed to listen: %+v", err)
            return
        } else {
            sctpListener = listener
        }

        logger.NgapLog.Infof("Listen on %s", sctpListener.Addr())

        for {
            newConn, err := sctpListener.AcceptSCTP(notimeout)
            if err != nil {
                if sctpListener.IsStopped() {
                    return
                }
                switch {
                case isRetryableSCTPError(err):
                    logger.NgapLog.Debugf("AcceptSCTP: %+v", err)
                default:
                    logger.NgapLog.Errorf("Failed to accept: %+v", err)
                }
                continue
            }
            // The new SCTP listener returns (nil, nil) when Close interrupts
            // AcceptSCTP. Treat that as a normal server shutdown.
            if newConn == nil {
                if sctpListener.IsStopped() {
                    return
                }
                logger.NgapLog.Error("AcceptSCTP returned a nil connection")
                continue
            }

            var info *sctp.SndRcvInfo
            if infoTmp, errGetDefaultSentParam := newConn.GetDefaultSentParam(); errGetDefaultSentParam != nil {
                logger.NgapLog.Errorf("Get default sent param error: %+v, accept failed", errGetDefaultSentParam)
                if errGetDefaultSentParam = newConn.Close(); errGetDefaultSentParam != nil {
                    logger.NgapLog.Errorf("Close error: %+v", errGetDefaultSentParam)
                }
                continue
            } else {
                info = infoTmp
                logger.NgapLog.Debugf("Get default sent param[value: %+v]", info)
            }

            info.PPID = ngapMessage.PPID
            if errSetDefaultSentParam := newConn.SetDefaultSentParam(info); errSetDefaultSentParam != nil {
                logger.NgapLog.Errorf("Set default sent param error: %+v, accept failed", errSetDefaultSentParam)
                if errSetDefaultSentParam = newConn.Close(); errSetDefaultSentParam != nil {
                    logger.NgapLog.Errorf("Close error: %+v", errSetDefaultSentParam)
                }
                continue
            } else {
                logger.NgapLog.Debugf("Set default sent param[value: %+v]", info)
            }

            events := sctp.SCTP_EVENT_DATA_IO | sctp.SCTP_EVENT_SHUTDOWN | sctp.SCTP_EVENT_ASSOCIATION
            if errSubscribeEvents := newConn.SubscribeEvents(events); errSubscribeEvents != nil {
                logger.NgapLog.Errorf("Failed to accept: %+v", errSubscribeEvents)
                if errSubscribeEvents = newConn.Close(); errSubscribeEvents != nil {
                    logger.NgapLog.Errorf("Close error: %+v", errSubscribeEvents)
                }
                continue
            } else {
                logger.NgapLog.Debugln("Subscribe SCTP event[DATA_IO, SHUTDOWN_EVENT, ASSOCIATION_CHANGE]")
            }

            if errSetReadBuffer := newConn.SetReadBuffer(int(readBufSize)); errSetReadBuffer != nil {
                logger.NgapLog.Errorf("Set read buffer error: %+v, accept failed", errSetReadBuffer)
                if errSetReadBuffer = newConn.Close(); errSetReadBuffer != nil {
                    logger.NgapLog.Errorf("Close error: %+v", errSetReadBuffer)
                }
                continue
            } else {
                logger.NgapLog.Debugf("Set read buffer to %d bytes", readBufSize)
            }

            if errSetReadTimeout := newConn.SetReadTimeout(readTimeout); errSetReadTimeout != nil {
                logger.NgapLog.Errorf("Set read timeout error: %+v, accept failed", errSetReadTimeout)
                if errSetReadTimeout = newConn.Close(); errSetReadTimeout != nil {
                    logger.NgapLog.Errorf("Close error: %+v", errSetReadTimeout)
                }
                continue
            } else {
                logger.NgapLog.Debugf("Set read timeout: %+v", readTimeout)
            }
            if errSetNoDelay := newConn.SetNoDelay(1); errSetNoDelay != nil {
                logger.NgapLog.Errorf("Set no delay error: %+v, accept failed", errSetNoDelay)
                if errSetNoDelay = newConn.Close(); errSetNoDelay != nil {
                    logger.NgapLog.Errorf("Close error: %+v", errSetNoDelay)
                }
                continue
            } else {
                logger.NgapLog.Debugf("Set SCTP_NODELAY to true")
            }
            logger.NgapLog.Infof("[AMF] SCTP Accept from: %+v", newConn.RemoteAddr())
            connections.Store(newConn, newConn)

            go handleConnection(newConn, readBufSize, handler)
        }
    }
    ```
    - build message
    ```bash
    cd free5gc
    make amf
    docker compose -f docker-compose-build.yaml build
    ```
- Factor 3
    - clone free5gc/openapi
    ```bash
    cd free5gc/base/free5gc
    git clone https://github.com/free5gc/openapi.git
    ```
    - modify go.mod (all NFs used free5gc/openapi), use amf as example (free5gc/base/free5gc/NFs/amf/go.mod)
    ```
    module github.com/free5gc/amf

    go 1.26.2

    require (
        github.com/asaskevich/govalidator v0.0.0-20230301143203-a9d515a09cc2
        github.com/davecgh/go-spew v1.1.1
        github.com/free5gc/nas v1.3.1-0.20261020065227-320605c34466
        github.com/free5gc/ngap v1.2.0
        github.com/free5gc/openapi v1.3.1-0.20261104070955-7fad8e868cb2
        github.com/free5gc/sctp v1.2.0
        github.com/free5gc/util v1.4.1-0.20261014070359-d865b0ce854b
        github.com/gin-gonic/gin v1.10.0
        github.com/google/uuid v1.6.0
        github.com/h2non/gock v1.2.0
        github.com/mitchellh/mapstructure v1.5.0
        github.com/mohae/deepcopy v0.0.0-20170929034955-c48cc78d4826
        github.com/pkg/errors v0.9.1
        github.com/prometheus/client_golang v1.21.0
        github.com/sirupsen/logrus v1.9.3
        github.com/stretchr/testify v1.10.0
        github.com/urfave/cli/v2 v2.27.7
        go.uber.org/mock v0.4.0
        gopkg.in/yaml.v2 v2.4.0
    )

    require (
        github.com/aead/cmac v0.0.0-20160719120800-7af84192f0b1 // indirect
        github.com/beorn7/perks v1.0.1 // indirect
        github.com/bytedance/sonic v1.11.6 // indirect
        github.com/bytedance/sonic/loader v0.1.1 // indirect
        github.com/cespare/xxhash/v2 v2.3.0 // indirect
        github.com/cloudwego/base64x v0.1.4 // indirect
        github.com/cloudwego/iasm v0.2.0 // indirect
        github.com/cpuguy83/go-md2man/v2 v2.0.7 // indirect
        github.com/felixge/httpsnoop v1.0.4 // indirect
        github.com/gabriel-vasile/mimetype v1.4.3 // indirect
        github.com/gin-contrib/sse v0.1.0 // indirect
        github.com/go-logr/logr v1.4.1 // indirect
        github.com/go-logr/stdr v1.2.2 // indirect
        github.com/go-playground/locales v0.14.1 // indirect
        github.com/go-playground/universal-translator v0.18.1 // indirect
        github.com/go-playground/validator/v10 v10.20.0 // indirect
        github.com/goccy/go-json v0.10.2 // indirect
        github.com/golang-jwt/jwt/v5 v5.2.2 // indirect
        github.com/h2non/parth v0.0.0-20190131123155-b4df798d6542 // indirect
        github.com/json-iterator/go v1.1.12 // indirect
        github.com/klauspost/compress v1.18.7 // indirect
        github.com/klauspost/cpuid/v2 v2.2.7 // indirect
        github.com/leodido/go-urn v1.4.0 // indirect
        github.com/mattn/go-isatty v0.0.20 // indirect
        github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd // indirect
        github.com/modern-go/reflect2 v1.0.2 // indirect
        github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
        github.com/pelletier/go-toml/v2 v2.2.2 // indirect
        github.com/pmezard/go-difflib v1.0.0 // indirect
        github.com/prometheus/client_model v0.6.1 // indirect
        github.com/prometheus/common v0.62.0 // indirect
        github.com/prometheus/procfs v0.15.1 // indirect
        github.com/russross/blackfriday/v2 v2.1.0 // indirect
        github.com/tim-ywliu/nested-logrus-formatter v1.3.2 // indirect
        github.com/twitchyliquid64/golang-asm v0.15.1 // indirect
        github.com/ugorji/go/codec v1.2.12 // indirect
        github.com/xrash/smetrics v0.0.0-20240521201337-686a1a2994c1 // indirect
        go.opentelemetry.io/contrib/instrumentation/net/http/httptrace/otelhttptrace v0.49.0 // indirect
        go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.49.0 // indirect
        go.opentelemetry.io/otel v1.24.0 // indirect
        go.opentelemetry.io/otel/metric v1.24.0 // indirect
        go.opentelemetry.io/otel/trace v1.24.0 // indirect
        golang.org/x/arch v0.8.0 // indirect
        golang.org/x/crypto v0.52.0 // indirect
        golang.org/x/net v0.55.0 // indirect
        golang.org/x/oauth2 v0.27.0 // indirect
        golang.org/x/sys v0.45.0 // indirect
        golang.org/x/text v0.37.0 // indirect
        google.golang.org/protobuf v1.36.1 // indirect
        gopkg.in/yaml.v3 v3.0.1 // indirect
    )
    replace github.com/free5gc/openapi => ../../openapi

    ```
    - update go mod
    ```bash
    cd free5gc/base/free5gc/NFs/amf
    go mod tidy
    ```
    - modify openapi
        - 
    ```go
    func GetTokenCtx(
        request TokenRequest,
    ) (context.Context, *models.ProblemDetails, error) {
        if err := request.validate(); err != nil {
            return nil, nil, err
        }
        tok, pd, err := sendAccTokenReq(request)
        if err != nil {
            return nil, pd, err
        }
        return context.WithValue(context.Background(),
            openapi.ContextOAuth2, tok), pd, nil
    }

    func sendAccTokenReq(
        request TokenRequest,
    ) (oauth2.TokenSource, *models.ProblemDetails, error) {
        cacheKey := tokenCacheKey(request)
        var client *AccTok.APIClient

        if val, ok := clientMap.Load(request.NRFURI); ok {
            client = val.(*AccTok.APIClient)
        } else {
            configuration := AccTok.NewConfiguration()
            configuration.SetBasePath(request.NRFURI)
            client = AccTok.NewAPIClient(configuration)
            clientMap.Store(request.NRFURI, client)
        }

        // Check if we have a valid cached token
        if val, ok := tokenMap.Load(cacheKey); ok {
            cached := val.(cachedToken)
            // Compare current time with absolute expiry timestamp
            if time.Now().Unix() < cached.ExpiryTime {
                token := &oauth2.Token{
                    AccessToken: cached.Response.Access_token,
                    TokenType:   cached.Response.Token_type,
                    Expiry:      time.Unix(cached.ExpiryTime, 0),
                }
                return oauth2.StaticTokenSource(token), nil, nil
            }
        }

        req := &AccTok.AccessTokenRequestRequest{}
        req.SetGrantType("client_credentials")
        req.SetNfInstanceId(request.ConsumerNFInstanceID)
        req.SetScope(request.Scope)
        if !isBlank(string(request.ConsumerNFType)) {
            req.SetNfType(request.ConsumerNFType)
        }
        if !isBlank(string(request.TargetNFType)) {
            req.SetTargetNfType(request.TargetNFType)
        }
        if !isBlank(request.TargetNFInstanceID) {
            req.SetTargetNfInstanceId(request.TargetNFInstanceID)
        }

        res, err := client.AccessTokenRequestApi.AccessTokenRequest(
            context.Background(), req)

        if err == nil {
            // TS 29.510 §6.3.4.2.2
            if res == nil || res.Nrf_AccTok_AccessTokenRsp == nil {
                return nil, nil, errors.New("invalid access token response: response is nil")
            }
            // TS 29.510 §6.3.5.2.3
            if isBlank(res.Nrf_AccTok_AccessTokenRsp.Access_token) {
                return nil, nil, errors.New("invalid access token response: access_token is empty")
            }
            // Calculate absolute expiry time: current time + expires_in seconds
            expiryTime := 1
            cached := cachedToken{
                Response:   *res.Nrf_AccTok_AccessTokenRsp,
                ExpiryTime: expiryTime,
            }
            tokenMap.Store(cacheKey, cached)

            token := &oauth2.Token{
                AccessToken: res.Nrf_AccTok_AccessTokenRsp.Access_token,
                TokenType:   res.Nrf_AccTok_AccessTokenRsp.Token_type,
                Expiry:      time.Unix(expiryTime, 0),
            }
            return oauth2.StaticTokenSource(token), nil, nil
        } else {
            return nil, nil, openapi.ReportError("server no response")
        }
    }
    ```
    - build message
    ```bash
    cd free5gc
    make
    docker compose -f docker-compose-build.yaml build
    ```

- Run Experiment
    - Control Plane
        
        run_control_plane_experiment.sh
        
    - User Plane
        
        run_user_plane_experiment.sh
        

## Acknowledgments

### **References**

- https://github.com/free5gc/free5gc
- https://github.com/aligungr/UERANSIM
- https://github.com/free5gc/gtp5g
- https://github.com/open5gs/open5gs
- https://github.com/PORVIR-5G-Project