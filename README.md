# distroless-all

## Tips

1. `nonroot` UID/GID 65532: https://github.com/GoogleContainerTools/distroless/blob/64ac73c84c72528d574413fb246161e4d7d32248/common/variables.bzl#L18

```dockerfile
ENV USER=nonroot
ENV UID=65532

RUN addgroup \
      --gid $UID \
      $USER && \
    adduser \
      --disabled-password \
      --gecos "" \
      --home "/nonexistent" \
      --shell "/sbin/nologin" \
      --no-create-home \
      --uid "${UID}" \
      --ingroup $USER \
      "${USER}"

# ...

RUN chown -R nonroot:nonroot /opt/var/cache
```

2. COPY executable files from normal builder to distroless image, consider about dependencies, r/w directories, required system files:

```dockerfile
# https://github.com/kyos0109/nginx-distroless/blob/4fa36b8c066303f34e490aad7b407d447ade4b7d/Dockerfile
FROM nginx as base

# https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
ARG TIME_ZONE

RUN sed -i -E 's/(listen[ ]+)80;/\18080;/' /etc/nginx/conf.d/default.conf

RUN mkdir -p /opt/bin /opt/etc /opt/usr/bin /opt/var/cache/nginx && \
    cp /usr/share/zoneinfo/${TIME_ZONE:-UTC} /opt/etc/localtime && \
    cp -a --parents /etc/passwd /opt && \
    cp -a --parents /etc/group /opt && \
    cp -aL --parents /var/run /opt && \
    cp -a --parents /usr/lib/nginx /opt && \
    cp -a --parents /usr/share/nginx /opt && \
    cp -a --parents /var/log/nginx /opt && \
    cp -a --parents /etc/nginx /opt && \
    cp -a --parents "$(which nginx)" /opt && \
    ldd "$(which nginx)" | grep -oP '(?<==> )/lib/[^ ]+\.so' | xargs -I {} bash -xc 'cp -a --parents {}* /opt' && \
    cp -a --parents "$(which nginx-debug)" /opt && \
    ldd "$(which nginx-debug)" | grep -oP '(?<==> )/lib/[^ ]+\.so' | xargs -I {} bash -xc 'cp -a --parents {}* /opt' && \
    true

RUN chown -R nonroot:nonroot /opt/var/cache /opt/var/run

RUN touch /opt/var/run/nginx.pid && \
    chown -R nonroot:nonroot /opt/var/run/nginx.pid

FROM gcr.io/distroless/base-debian12:nonroot

COPY --from=base /opt /

EXPOSE 8080 8443

ENTRYPOINT ["/usr/sbin/nginx", "-g", "daemon off;"]
```

## healthcheck

> https://github.com/GoogleContainerTools/distroless/issues/183

1. Use the runtime in the distroless image:

> https://github.com/GoogleContainerTools/distroless/issues/1350#issuecomment-1619276371  
> https://www.mattknight.io/blog/docker-healthchecks-in-distroless-node-js

```yaml
healthcheck:
  test:
    [
      "CMD",
      "node",
      "-e",
      "require('http').get('http://localhost:3000/api/profile', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})",
    ]
```

2. embed the healthcheck to the PID 1, Go for example:

```go
	healthcheck := flag.String("healthcheck", "", "http://1.example.org,http://2.example.org")
	flag.Parse()
	if *healthcheck != "" {
		http.DefaultClient.Timeout = time.Second
		for _, healhealthcheckURL := range strings.Split(*healthcheck, ",") {
			if healhealthcheckURL == "" {
				continue
			}
			if _, err := http.Get(healhealthcheckURL); err != nil {
				os.Exit(1)
			}
		}
		os.Exit(0)
	}
```

3. import a third-party tiny executable

> https://github.com/GoogleContainerTools/distroless/issues/183#issuecomment-1483015906

```dockerfile
FROM busybox AS builder

ARG BUSYBOX_VERSION=1.31.0-i686-uclibc
ADD https://busybox.net/downloads/binaries/$BUSYBOX_VERSION/busybox_WGET /wget
RUN chmod a+x /wget

FROM gcr.io/distroless/java

COPY --from=builder /wget /usr/bin/wget
```
