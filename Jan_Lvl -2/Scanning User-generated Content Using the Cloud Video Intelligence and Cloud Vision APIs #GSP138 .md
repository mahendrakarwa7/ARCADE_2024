# GSP138
## Run in cloudshell
### Get region from Task 6 as shown in screenshot
```cmd
export REGION=
```
![hint](https://cdn.discordapp.com/attachments/1153622026288910428/1166324411587108875/hint.png)
```cmd
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export IV_BUCKET_NAME=${PROJECT_ID}-upload
export FILTERED_BUCKET_NAME=${PROJECT_ID}-filtered
export FLAGGED_BUCKET_NAME=${PROJECT_ID}-flagged
export STAGING_BUCKET_NAME=${PROJECT_ID}-staging
gsutil mb gs://${IV_BUCKET_NAME}
gsutil mb gs://${FILTERED_BUCKET_NAME}
gsutil mb gs://${FLAGGED_BUCKET_NAME}
gsutil mb gs://${STAGING_BUCKET_NAME}
export UPLOAD_NOTIFICATION_TOPIC=upload_notification
gcloud pubsub topics create ${UPLOAD_NOTIFICATION_TOPIC}
gcloud pubsub topics create visionapiservice
gcloud pubsub topics create videointelligenceservice
gcloud pubsub topics create bqinsert
gsutil notification create -t upload_notification -f json -e OBJECT_FINALIZE gs://${IV_BUCKET_NAME}
gsutil -m cp -r gs://spls/gsp138/cloud-functions-intelligentcontent-nodejs .
cd cloud-functions-intelligentcontent-nodejs
export DATASET_ID=intelligentcontentfilter
export TABLE_NAME=filtered_content
bq --project_id ${PROJECT_ID} mk ${DATASET_ID}
bq --project_id ${PROJECT_ID} mk --schema intelligent_content_bq_schema.json -t ${DATASET_ID}.${TABLE_NAME}
sed -i "s/\[PROJECT-ID\]/$PROJECT_ID/g" config.json
sed -i "s/\[FLAGGED_BUCKET_NAME\]/$FLAGGED_BUCKET_NAME/g" config.json
sed -i "s/\[FILTERED_BUCKET_NAME\]/$FILTERED_BUCKET_NAME/g" config.json
sed -i "s/\[DATASET_ID\]/$DATASET_ID/g" config.json
sed -i "s/\[TABLE_NAME\]/$TABLE_NAME/g" config.json
gcloud functions deploy GCStoPubsub --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic ${UPLOAD_NOTIFICATION_TOPIC} --entry-point GCStoPubsub --region $REGION
gcloud functions deploy visionAPI --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic visionapiservice --entry-point visionAPI --region $REGION
gcloud functions deploy videoIntelligenceAPI --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic videointelligenceservice --entry-point videoIntelligenceAPI --timeout 540 --region $REGION
gcloud functions deploy insertIntoBigQuery --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic bqinsert --entry-point insertIntoBigQuery --region $REGION
curl -o cloudhustlers.jpg https://cdn.discordapp.com/attachments/1153622026288910428/1153626032692269137/2000x700headlineimg.png
gsutil cp cloudhustlers.jpg gs://$IV_BUCKET_NAME/
echo "
#standardSql

SELECT insertTimestamp,
  contentUrl,
  flattenedSafeSearch.flaggedType,
  flattenedSafeSearch.likelihood
FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
CROSS JOIN UNNEST(safeSearch) AS flattenedSafeSearch
ORDER BY insertTimestamp DESC,
  contentUrl,
  flattenedSafeSearch.flaggedType
LIMIT 1000
" > sql.txt
sleep 30
bq --project_id ${PROJECT_ID} query < sql.txt
```
