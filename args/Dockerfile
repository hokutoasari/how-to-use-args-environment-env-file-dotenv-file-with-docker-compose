FROM busybox

ARG ARG_VALUE1="arg1_in_Dockerfile."
ARG ARG_VALUE2="arg2_in_Dockerfile."

RUN echo "ARG_VALUE1 is ${ARG_VALUE1}" \
  && echo "ARG_VALUE2 is ${ARG_VALUE2}" \
  && echo "ARG_VALUE3 is ${ARG_VALUE3}" \
  && echo "ARG_VALUE4 is ${ARG_VALUE4}"
RUN echo "ARG_VALUE1 is $ARG_VALUE1" >> /tmp/outs.txt \
  && echo "ARG_VALUE2 is $ARG_VALUE2" >> /tmp/outs.txt \
  && echo "ARG_VALUE3 is $ARG_VALUE3" >> /tmp/outs.txt \
  && echo "ARG_VALUE4 is $ARG_VALUE4" >> /tmp/outs.txt

CMD cat /tmp/outs.txt \
  && echo "-----" \
  && echo "ARG_VALUE1 is ${ARG_VALUE1}" \
  && echo "ARG_VALUE2 is ${ARG_VALUE2}" \
  && echo "ARG_VALUE3 is ${ARG_VALUE3}" \
  && echo "ARG_VALUE4 is ${ARG_VALUE4}"
