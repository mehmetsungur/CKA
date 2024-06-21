1. context komutu çok önemli.

2. cronjob oluşturma.
Jobs history limits
The .spec.successfulJobsHistoryLimit and .spec.failedJobsHistoryLimit fields are optional.
Deadline for delayed job start
The .spec.startingDeadlineSeconds field is optional. This field defines a deadline (in whole seconds) for starting the Job, if that Job misses its scheduled time for any reason.
cronjob'dan job oluşturma
  # Create a job from a cron job named "a-cronjob"
  kubectl create job test-job --from=cronjob/a-cronjob

3. deployment hata.
deployment verilmiş.
apiversion: apps/v1
selector eklemek gerekti.

4. limitrange
memory limitini namespace'in max limitinin yarısı yap.
request .. memory olsun. 

5. secret , configmap

6. Dockerfile.
Şu isim şu tagle image oluştur. İstediğin container engine kullanabilirsin.

7. ingress sorusu.
deployment, service ve ingress var.
deploymneta dokunma.
Service'in selector kısmı düzeltildi. (kubectl get ep)
ingress'de service ismi ve path yanlıştı. Düzeltildi.
Host eklendi.

8. maxsurge,maxavailable
image update et.
kubectl rollout undo deploy deploy-name

9. pod, deployment oluştur. ad, image , label

10. serviceaccountname yaz deployment'ın pod kısmına

11. RBAC
logad podları görmüyor. Podun pod görme yetkisi yok.
İlgili serviceaccount'u bahse konu poda eklenmesine.

12. network policy
netpollar hazır.
Sen sadece sana verilen podu bu netpolleri kullanacak şekilde update et.
netpoldaki podselector labellarını verilen poda ekle.

13. resource (request ve limit)

14. Sadece env. variable ekle.

15. security context ekle.
runAsUser: 2000 olsun ve
allowPrivilegeEscalation: false olsun.

16. readiness prob sorusu