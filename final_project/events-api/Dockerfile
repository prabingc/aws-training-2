FROM public.ecr.aws/docker/library/node:slim

COPY . /api/

WORKDIR /api

RUN npm install

CMD ["node", "server.js"]
