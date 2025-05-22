FROM public.ecr.aws/docker/library/node:slim

COPY . /web/

WORKDIR /web

RUN npm install

CMD ["npm", "start"]
