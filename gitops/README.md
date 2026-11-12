# Dell OptiPlex: primeiro deploy GitOps no Argo CD

Pré-requisitos: K3s, Argo CD, Traefik e MetalLB já configurados. Traefik em 192.168.3.230. Repositório Git acessível ao Argo CD.

## Estrutura
- `gitops/apps/nginx`: Deployment, Service ClusterIP e Ingress para `nginx.home.arpa`.
- `gitops/bootstrap/nginx-application.yaml`: Application do Argo CD, apontando ao repositório Git.

## Publicar
1. Copie a pasta `gitops/` para a raiz do seu repositório Git, faça commit e push na branch `main`.
2. Edite `gitops/bootstrap/nginx-application.yaml` e substitua `REPLACE_GIT_REPO_URL` pela URL real do seu repositório (por exemplo `https://github.com/USUARIO/dell-optiplex.git`). Se usar outra branch ou caminho, ajuste `targetRevision`/`path`.
3. Faça commit e push novamente. Se o repositório for privado, cadastre-o previamente em Argo CD > Settings > Repositories, com credenciais de leitura.
4. Execute no Dell:
   ```bash
   sudo k3s kubectl apply -f gitops/bootstrap/nginx-application.yaml
   sudo k3s kubectl get applications -n argocd
   sudo k3s kubectl get deploy,svc,ingress -n demo
   ```
   Se o Git estiver só na sua máquina de desenvolvimento, copie o arquivo bootstrap para o Dell ou execute kubectl no computador com acesso ao cluster.
5. No Windows, adicione ao arquivo `C:\Windows\System32\drivers\etc\hosts`:
   `192.168.3.230 nginx.home.arpa`
6. Teste sem DNS configurado: `curl -H 'Host: nginx.home.arpa' http://192.168.3.230/`.

## Testar GitOps
- Mude `replicas: 1` para `replicas: 2` em `gitops/apps/nginx/deployment.yaml`, faça commit e push; verifique `sudo k3s kubectl get deployment nginx -n demo` e a UI do Argo CD.
- Para testar Self-Heal, rode `sudo k3s kubectl scale deployment nginx -n demo --replicas=3` e observe o retorno a 2 após a reconciliação.
- `prune: true` remove recursos que forem retirados do Git. Não use este projeto para controlar recursos preexistentes sem revisar seus ownerships.

## Observações
- Nginx usa Service `ClusterIP`: **não** consome nenhum outro IP do MetalLB; a entrada é pelo Traefik em `192.168.3.230`.
- O HTTP aqui é deliberado para o laboratório; não exponha o serviço diretamente à internet sem TLS e controles adicionais.
- Antes de aplicar, confira que não existe outro Ingress `nginx.home.arpa` gerenciado por testes anteriores. Se existir, remova ou migre o recurso para este app.
- O sincronismo automático do Argo CD não é instantâneo; depende de reconciliação ou webhook.
