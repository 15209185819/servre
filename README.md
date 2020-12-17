# servre
#include <sys/time.h>
#include "bstp.h"
#include "innerFormat.h"
#include "stmSrvLog.h"
#include "ReadFile.h"
#include <assert.h>
#include "jsonWrapper.h"
#include "configini.h"

static int transHeadBstar2Inner(BSTPHeader *pBs, InnerDataHeader *pInn)
{
    memcpy(&pInn->bstpH, pBs, sizeof(pInn->bstpH));
    return 0;
};

int createTransEleStm(ITransEleStm** ppv)
{
	CReadFile*  p = new CReadFile();
	ITransEleStm* pv = p;
	*ppv = pv;

	return 0;
}
struct Cfg
{
    std::string strFormat;//264 265 m4e
    std::string cfgDir;
};

int InitByCfg(const char *fullfilename);

static Cfg *g_cfg = NULL;
//===================================================================================================================
CReadFile::CReadFile()
{
    m_bExit = false;
    m_pDataThd = NULL;
};
CReadFile::~CReadFile()
{
    stmSrvlogInfo("in,streamId: %u", m_streamId);
    stop();
	stmSrvlogInfo("out,streamId: %u",  m_streamId);
};
 
int CReadFile::run()
{
    if(g_cfg==NULL)
    {
        g_cfg = new Cfg;                                                                    //文件的数据类型
        g_cfg->cfgDir = getProperty("cfgDir");                                              //获取文件的路径
        //getProperty("logDir")
    }
    std::string fullcfgname = g_cfg->cfgDir + "/" + m_modeName + ".conf";                   //加上配置文件路径
    InitByCfg(fullcfgname.c_str());     
    m_pDataThd = new std::thread( [=] { _getDataThread(); } );

    return 0;
}

 
int CReadFile::stop()
{
    

    if ( m_pDataThd != NULL )
    {
		stmSrvlogInfo("stop '%s'", typeid(*this).name());
        m_bExit = true;
        if (m_pDataThd->joinable())
            m_pDataThd->join();
        delete ( m_pDataThd );
        m_pDataThd = NULL;
    }

    return 0;
}

static int get_next_video_frame2(uint8_t **buff, int *size, int* nStartCode,FILE *fp)
{
    uint8_t szbuf[1024];
    int szlen = 0;
    int ret;
    *size = 0;

    while ((ret = fread(szbuf + szlen, 1, sizeof(szbuf) - szlen, fp)) > 0)      //读取文件数据并分离出每帧
    {
        int i = 3;
        szlen += ret;
        while (i < szlen - 3 && !(szbuf[i] == 0 && szbuf[i + 1] == 0 && (szbuf[i + 2] == 1 || (szbuf[i + 2] == 0 && szbuf[i + 3] == 1))))
            i++;
        if ((*nStartCode) == 0 && (szbuf[i + 2] == 1 || (szbuf[i + 2] == 0 && szbuf[i + 3] == 1)))
        {
            if (szbuf[i + 2] == 1)
            {
                *nStartCode = 3;
            }
            else if ((szbuf[i + 2] == 0 && szbuf[i + 3] == 1))
            {
                *nStartCode = 4;
            }
            stmSrvlogWarn("nStartCode:%d\n", *nStartCode);
        }
        memcpy(*buff + *size, szbuf, i);                                //将szbuf内容拷贝到*buff地址向后移动*size个位置上
        *size += i;
        memmove(szbuf, szbuf + i, szlen - i);
        szlen -= i;
        if (szlen > 3)
        {
            //printf("szlen %d\n", szlen);
            fseek(fp, -szlen, SEEK_CUR);                                       //从当前位置向前读取szlen个字符
            break;
        }
    }
    if (ret > 0)
        return *size;
    fseek(fp, 0, SEEK_SET);
    return 0;
}
static uint64_t os_gettime2(void)
{
#ifdef WIN32
    return (timeGetTime() * 1000ll);
#else
    struct timespec tp;
    clock_gettime(CLOCK_MONOTONIC, &tp);
    return (tp.tv_sec * 1000000llu + tp.tv_nsec / 1000llu);
#endif
}
void CReadFile::_getDataThread()
{
    std::string streamToken = getProperty("streamToken");
    Json::Value root;

    if (true != JsonParse(streamToken).toJson(root))
    {
        stmSrvlogErr("parseJson err. streamToken:%s\n", streamToken.c_str());
        return;
    }
    //streamToken:{"chn":1,"playtype":0,"stype":1}
    if (!root.isMember("chn") || !root["chn"].isInt() || !root.isMember("playtype") || !root["playtype"].isInt() || !root.isMember("stype") || !root["stype"].isInt())
    {
        stmSrvlogErr("2 parseJson err. streamToken:%s\n", streamToken.c_str());
        return;
    }
    stmSrvlogInfo("play streamToken:%s", streamToken.c_str());
    int nChn = root["chn"].asInt();                                                          //????内容格式转换成int
    int sType = root["stype"].asInt();
    int playtype = root["playtype"].asInt();

    m_bExit = false;
    static uint8_t *vbuf = NULL;
    std::unique_ptr<char> cachePtr(new char[1024 * 1000]);
    uint32_t sequence = 0;
    uint64_t ts = 0;
    ts = os_gettime2();
    FILE *fp = NULL;
    if (fp == NULL)
    {
        char strProcessPath[1024] = {0};
        char *strProcessName = NULL;
        char *pDirEnd = NULL;
        assert(readlink("/proc/self/exe", strProcessPath, sizeof(strProcessPath) - 1) > 0);   //判断将路径拷贝到 strProcessPath是否成功
        assert((strProcessName = strrchr(strProcessPath, '/')) != NULL && (pDirEnd = strProcessName) && strProcessName++);
        *pDirEnd = '\0';                                                                      //末尾加上结束符
        std::string filename = std::string(strProcessPath) + "/test."+g_cfg->strFormat;

        fp = fopen(filename.c_str(), "rb");                                    //
        if (!fp)
        {
            stmSrvlogErr("open %s failed\n", filename.c_str());
            return ;                                                              //return -1;
        }
        if (!vbuf)
        {
            vbuf = (uint8_t *)malloc(2 * 1024 * 1024);
            if (!vbuf)
            {
                stmSrvlogErr( "malloc %s failed\n", filename.c_str());
                return ;                                                          //return -1;
            }
        }
    }
    int nStartCode = 0;
    while (!m_bExit)
    {
        BSTPHeader BsHeader;
        memset(&BsHeader, 0, sizeof(BsHeader));
        BsHeader.mark = 0xBF9D1FDB;
        BsHeader.type = 2; //2=video
        BsHeader.sequence = sequence++;        
        struct timeval val;
        gettimeofday(&val,NULL);
        BsHeader.timeStamp = val.tv_sec;
        BsHeader.tick = val.tv_usec;
        BsHeader.format[0] = 4;//264

        int nCache = 0;
        char *p = cachePtr.get();
        BSTPHeader* pBsHeader = (BSTPHeader *)p;
        memcpy(p + nCache, &BsHeader, sizeof(BSTPHeader));
        nCache += sizeof(BSTPHeader);

    chn0_video_again:                                                                         //goto location
        int vsize = 0;
        int ret = get_next_video_frame2(&vbuf, &vsize, &nStartCode, fp);
        if (ret < 0)
        {
            fprintf(stderr, "get_next_video_frame failed\n");
            return;
        }
        if (ret == 0)
        {
            stmSrvlogWarn("end of file,go on next turn, have send cnt:%d", sequence);
            goto chn0_video_again;
        }
        if(g_cfg->strFormat=="264")
        {
            pBsHeader->format[0] = 4;
            int nal_type = vbuf[nStartCode] & 0x1F;                                              //0001 1111 & code (与0x01=1)
            if (nal_type == 1) //p frame
            {
                pBsHeader->format[3] = 3;//P
                memcpy(cachePtr.get() + nCache, vbuf, vsize);
                nCache += vsize;
            }
            else
            {
                char *p = cachePtr.get();
                memcpy(p + nCache, vbuf, vsize);
                nCache += vsize;
                if (nal_type != 5)
                {
                    //stmSrvlogWarn("try to find nal_type5");
                    goto chn0_video_again;
                }
                else
                {
                    pBsHeader->format[3] = 1;//I
                }
                
            }           
        }
        else
        {
            if (g_cfg->strFormat == "265")
            {
                pBsHeader->format[0] = 5;
            }
            else if(g_cfg->strFormat=="m4e")
            {
                pBsHeader->format[0] = 1;
            }
            memcpy(cachePtr.get() + nCache, vbuf, vsize);
            nCache += vsize;
        }

        if (m_cbDataInFunc)
        {
            pBsHeader->length = nCache;            

            InnerDataHeader innerHead;
            transHeadBstar2Inner(pBsHeader, &innerHead);                                          //加Bstr头
           // stmSrvlogWarn("in cb len: %d", innerHead.bstpH.length);
            m_cbDataInFunc(&innerHead, (const char *)cachePtr.get(), nCache + sizeof(innerHead), m_cbDataInParam);//回调出去
            //stmSrvlogWarn("out cb len: %d", innerHead.bstpH.length);
        }
        //ES_PS_RTP_Send((const char *)cachePtr.get(), nCache);
        while (os_gettime2() - ts < 1000000 / 25)
        {
            //printf(".........\n");
            usleep(1000);
        }
        ts += 1000000 / 25;
    }
    if (fp)
    {
        fclose(fp);
        fp = NULL;
    }

    stmSrvlogWarn( "streamId: %u thread exit", m_streamId );

    return;
}

static int Load(const char *szFullname)
{                                                                               
    if (access(szFullname, F_OK) != 0)                                            //如果文件不存在，则创建文件，并生成默认值
    {
        fprintf(stderr, "path = %s not exist, will create default \n", szFullname);
        char cmd[256] = {0};
        snprintf(cmd, 256, "echo '[ReadFile]' > %s", szFullname);
        system(cmd);
        memset(cmd, 0, 256);
        snprintf(cmd, 256, "echo '#choose in [264,265,m4e,etc...]' >> %s", szFullname);
        system(cmd);
        /*
        [ReadFile]
        #choose in [264,265,m4e,etc...]
        format=264
        */

        memset(cmd, 0, 256);
        snprintf(cmd, 256, "echo 'format=264' >> %s", szFullname);
        system(cmd);

    }
    if (access(szFullname, F_OK) != 0)
    {
        fprintf(stderr, "path = %s.conf   create failed \n", szFullname);
        return -1;
    }

    return 0;
}
int InitByCfg(const char *fullfilename)
{
    std::string cfgFileName = std::string(fullfilename);
    stmSrvlogInfo("cfgFileName:%s",cfgFileName.c_str());
    if (0 != Load(cfgFileName.c_str()))                                                 //打开文件加载数据
    {
        return -1;
    }

    Config *cfg = NULL;
    if (ConfigReadFile(cfgFileName.c_str(), &cfg) != CONFIG_OK)                                 //????读取文件配置
    {
        stmSrvlogErr(" ConfigOpenFile failed for %s", cfgFileName.c_str());
        return -1;
    }
    int nRet = 0;
    const char *sect = "ReadFile";
 
    {
        char val[1024] = {0};
        const char *key = "format";
        if (ConfigReadString((Config *)cfg, sect, key, val, 1024, "264") != CONFIG_OK)                  //????
        {
            stmSrvlogErr("not find,sect =%s, key =%s, val=%s,will use default\n", sect, key, val);
        }
        if(g_cfg)
        {
            g_cfg->strFormat = val;

        }
    }
    stmSrvlogInfo("use format:%s", g_cfg->strFormat.c_str());

    nRet = 0;

EXIT:
    if (cfg)
    {
        ConfigFree((Config *)cfg);                                                            //????
        cfg = NULL;
    }

    return nRet;
}
