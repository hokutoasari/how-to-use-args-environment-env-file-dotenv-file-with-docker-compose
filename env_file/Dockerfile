FROM busybox

ENV ENV_VALUE1="env1_in_Dockerfile."
ENV ENV_VALUE2="env2_in_Dockerfile."

RUN echo "ENV_VALUE1 is ${ENV_VALUE1}" \
  && echo "ENV_VALUE2 is ${ENV_VALUE2}" \
  && echo "ENV_VALUE3 is ${ENV_VALUE3}" \
  && echo "ENV_VALUE4 is ${ENV_VALUE4}" \
  && echo "ENV_VALUE5 is ${ENV_VALUE5}"
RUN echo "ENV_VALUE1 is $ENV_VALUE1" >> /tmp/outs.txt \
  && echo "ENV_VALUE2 is $ENV_VALUE2" >> /tmp/outs.txt \
  && echo "ENV_VALUE3 is $ENV_VALUE3" >> /tmp/outs.txt \
  && echo "ENV_VALUE4 is $ENV_VALUE4" >> /tmp/outs.txt \
  && echo "ENV_VALUE5 is $ENV_VALUE5" >> /tmp/outs.txt

CMD cat /tmp/outs.txt \
  && echo "-----" \
  && echo "ENV_VALUE1 is ${ENV_VALUE1}" \
  && echo "ENV_VALUE2 is ${ENV_VALUE2}" \
  && echo "ENV_VALUE3 is ${ENV_VALUE3}" \
  && echo "ENV_VALUE4 is ${ENV_VALUE4}" \
  && echo "ENV_VALUE5 is ${ENV_VALUE5}"
