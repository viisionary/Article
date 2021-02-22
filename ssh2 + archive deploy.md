# ssh2 +  archive 完成前端项目部署

### build 文件打包

> npm install archiver —save
>
> npm install fs-extra —save

```jsx
const chalk = require('chalk');
const { appBuild, appPath } = require('../config/paths');
const fsExtra = require('fs-extra');
const paths = require('../config/paths');
const appPublic = require(paths.appPublic + '/manifest.json');
let archiver = require('archiver')
const version = appPublic.version;
let fs = require('fs')
const project = appPublic.name;

const doPackage = async () => {
	try {
		// dir name `${project}-${version}`
		await fsExtra.copy(appBuild, `${appPath}/${project}-${version}`);
		let output = fs.createWriteStream(`${appPath}/${project}-${version}.tar`)
		let archive = archiver('tar', {
			zlib: { level: 9 } // 设置压缩级别
		})
		archive.pipe(output);
		// 归档 `${project}-${version}.tar`
		archive.directory(`${appPath}/${project}-${version}`,`${project}-${version}`);
		output.on('close', function() {
			console.log(`总共 ${chalk.green(parseInt(archive.pointer()/(1024*1024)))} MB`)
			console.info(chalk.green('Package Succeed ！'));
		})
		archive.on('error', function(err) {
			throw err
		})
		await archive.finalize();

	} catch (e) {
		console.error(chalk.red('Package with warnings.\n'));
		console.error(e);
	}
};

doPackage();
```

### tar 文件上传

> npm install ssh2 —save

```jsx
const chalk = require('chalk');
const ora = require('ora');
const { Client } = require('ssh2');
const paths = require('../config/paths');
const appPublic = require(paths.appPublic + '/manifest.json');
const appPath = paths.appPath;
const { getHostIp }  = require('./generateHost')
const os = require('os');

const username = 'ccbop';
const HostConfig = {
	host: getHostIp(), // 服务器地址
	port: 19222,
	username,
	password: 'pwd'

};
// 存放tar路径
const serverBaseCodePath = `/home/${username}/wwwRoot/code/`;
// nginx项目目录
const serverBaseSitePath = `/home/${username}/wwwRoot/ccbop-frontend/`;
const project = appPublic.name;
const version = appPublic.version;

// tar 文件名【兼容 Windows】
const localTarPath = os.type() ==='Darwin'?`${appPath}/${project}-${version}.tar`:`${appPath}\\${project}-${version}.tar`;
// const serverTarPath = '';

// tar位置
const serverTarPath = serverBaseCodePath + project + '-' + version + '.tar';
// tar 解压位置
const serverCodePath = serverBaseCodePath + project + '-' + version;

const doDeploy = () => {
	/**
	 *  code path
	 *  /site/code/main-app
	 *  build path
	 *  /site/main-app
	 */

	const conn = new Client();
	conn.on('ready', () => {
		conn.sftp((err, sftp) => {
			const loading = ora('tar uploading...');
			loading.start();
			if (err) throw err;
			sftp.fastPut(localTarPath, serverTarPath, {
				chunkSize: 65535 * 1024 * 1024,
				concurrency: 128,
				step(total_transferred, chunk, total) {
					console.log(total_transferred, total);
					if (total_transferred < total) {
					}
				}
			}, (err) => {
				if (err) throw err;
				conn.exec(`rm -rf ${serverBaseSitePath}${project}/* && tar -xvf ${serverTarPath} -C ${serverBaseCodePath} && cp -r ${serverCodePath}/* ${serverBaseSitePath}${project}/`, function (err1, stream) {
					loading.stop();
					endLog(stream, conn)
				});
			});
		});
	}).connect(HostConfig);
};
doDeploy();

function endLog(stream, conn) {
	stream.on('close', function (code, signal) {
		console.log(chalk.cyan(`${project} ${HostConfig.host} put completed!\n`));
		conn.end();
	})
		.on('data', function (data) {
			console.log(chalk.green('OUTPUT: ') + data)
		});
}
```
