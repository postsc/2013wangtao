# 内存播放器（Cap2Disp）

标签（空格分隔）： V4L2 SDL2.0 capture

---

``` C
/**
 * Function of this file: 
 * 	-1. Capture raw video frome UVC
 * 	-2. And then display data in time using Simple Direct Layer Version 2.0.3
 * 	-3. After capturing 200 frames, there will be a pause. It's weired.
 * Version: 0.1;
 * Date: 2015-03-16;
 * Author: Lor(a.k.a ButcherCat);
 * E-mail: wt_lor@163.com
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <assert.h>

#include <getopt.h>

#include <fcntl.h>              /* low-level i/o */
#include <unistd.h>
#include <errno.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/time.h>
#include <sys/mman.h>
#include <sys/ioctl.h>

#include <asm/types.h>		// for videodev2.h
#include <linux/videodev2.h>

/* For SDL2*/
#include <SDL2/SDL.h>
#include <SDL2/SDL_thread.h>

#define CLEAR(x) memset (&(x), 0, sizeof (x))
#define WIDTH 640
#define HEIGHT 480

/*For SDL2*/
#define REFRESH_EVENT	(SDL_USEREVENT + 1)

/* Some variables for capturing*/
static char  yuv420p[WIDTH * HEIGHT * 3 >> 1];
FILE *fp;

struct buffer{
	void *	start;
	size_t	length;
};

static char *dev_name = NULL;
static int fd = -1;
struct buffer *buffers = NULL;
static unsigned int n_buffers = 0;

/* Some variables for SDL2*/
int screen_w = 800;
int screen_h = 600;

const int pix_w = WIDTH;
const int pix_h = HEIGHT;

SDL_Texture *texture;
SDL_Renderer *renderer;
SDL_Rect rect;
SDL_Window *screen;

/* Some funcitons for SDL2*/
int initialize_SDL(void)
{
	if(SDL_Init(SDL_INIT_VIDEO) < 0){
		fprintf(stderr, "Cannot initialize:%s\n", SDL_GetError());
		return -1;
	}

	return 1;
}

/* Some functions for capturing*/
static void errno_exit(const char *s)
{
	fprintf(stderr, "%s error %d, %s\n", s, errno, strerror(errno));

	exit(EXIT_FAILURE);
}

static int xioctl(int fd,int request,void *arg)
{
        int r;

        do r = ioctl (fd, request, arg);
        while (-1 == r && EINTR == errno);

        return r;
}

/* process_image() will convert yuv422 packaged into
 * yuv420p planar. You should know original video data
 * is yuv422 packaged.*/
static void process_image(const char *p, char *yuv420p, FILE *fp)
{
	char *y = yuv420p;
	char *u = &yuv420p[WIDTH * HEIGHT];
	char *v = &yuv420p[WIDTH * HEIGHT + WIDTH * HEIGHT / 4];

	int i, j;

	for(j = 0; j < HEIGHT; j++){
		for(i = 0 ; i < WIDTH * 2; i++){
			if(i % 2 == 0){
				*y = p[i+j*WIDTH*2];
				++y;
			}	

			if(j % 2 == 0){
				if(i % 4 == 1){
					*u = p[j*WIDTH*2 + i];
					++u;
				}	
			}

			if(j % 2 == 1){
				if(i % 4 == 3){
					*v = p[j*WIDTH*2 + i];
					++v;
				}	
			}
		}	
	}

//	fwrite(yuv420p,WIDTH * HEIGHT * 3 >> 1,1,fp);

	SDL_UpdateTexture(texture, NULL, yuv420p, pix_w);

	SDL_RenderClear(renderer);
	SDL_RenderCopy(renderer, texture, NULL, &rect);
	SDL_RenderPresent(renderer);
	
	SDL_Delay(40);
}

static int read_frame(void)
{
	struct v4l2_buffer buf;
	unsigned int i;

	CLEAR (buf);

    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_MMAP;

    if (-1 == xioctl (fd, VIDIOC_DQBUF, &buf)) {
           switch (errno) {
            	case EAGAIN:
                    		return 0;

				case EIO:
				/* Could ignore EIO, see spec. */

				/* fall through */

				default:
				errno_exit ("VIDIOC_DQBUF");
			}
	}

    assert (buf.index < n_buffers);

	process_image (buffers[buf.index].start, yuv420p, fp);

	if (-1 == xioctl (fd, VIDIOC_QBUF, &buf))
			errno_exit ("VIDIOC_QBUF");
	
	return 1;
}

static void mainloop(void)
{
	unsigned int count;

//        count = 200;
	count = 1;

        while (count != 0) {
                for (;;) {
                        fd_set fds;
                        struct timeval tv;
                        int r;

                        FD_ZERO (&fds);
                        FD_SET (fd, &fds);

                        /* Timeout. */
                        tv.tv_sec = 2;
                        tv.tv_usec = 0;

                        r = select (fd + 1, &fds, NULL, NULL, &tv);

                        if (-1 == r) {
                                if (EINTR == errno)
                                        continue;

                                errno_exit ("select");
                        }

                        if (0 == r) {
                                fprintf (stderr, "select timeout\n");
                                exit (EXIT_FAILURE);
                        }

			if (read_frame ())
                    		break;
	
			/* EAGAIN - continue select loop. */
                }

//	count--;
        }
}

static void stop_capturing(void)
{
	enum v4l2_buf_type type;

	type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

	if(-1 == xioctl(fd, VIDIOC_STREAMOFF, &type))
		errno_exit("VIDIOC_STREAMOFF");

}

static void start_capturing(void)
{
	unsigned int i;
	enum v4l2_buf_type type;

	for (i = 0; i < n_buffers; ++i) {
        struct v4l2_buffer buf;

        CLEAR (buf);

        buf.type        = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory      = V4L2_MEMORY_MMAP;
        buf.index       = i;

        if (-1 == xioctl (fd, VIDIOC_QBUF, &buf))
                    		errno_exit ("VIDIOC_QBUF");
	}
		
	type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

	if (-1 == xioctl (fd, VIDIOC_STREAMON, &type))
		errno_exit ("VIDIOC_STREAMON");
}

static void uninit_device(void)
{
	unsigned int i;

	for (i = 0; i < n_buffers; ++i)
			if (-1 == munmap (buffers[i].start, buffers[i].length))
				errno_exit ("munmap");

			free(buffers);
}

static void init_mmap(void)
{
	struct v4l2_requestbuffers req;

    CLEAR (req);

    req.count               = 4;
    req.type                = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    req.memory              = V4L2_MEMORY_MMAP;

	if (-1 == xioctl (fd, VIDIOC_REQBUFS, &req)) {
            if (EINVAL == errno) {
                    fprintf (stderr, "%s does not support "
                             "memory mapping\n", dev_name);
                    exit (EXIT_FAILURE);
            } else {
                    errno_exit ("VIDIOC_REQBUFS");
            }
    }

    if (req.count < 2) {
            fprintf (stderr, "Insufficient buffer memory on %s\n",
                     dev_name);
            exit (EXIT_FAILURE);
    }

    buffers = calloc (req.count, sizeof (*buffers));

    if (!buffers) {
            fprintf (stderr, "Out of memory\n");
            exit (EXIT_FAILURE);
    }

    for (n_buffers = 0; n_buffers < req.count; ++n_buffers) {
            struct v4l2_buffer buf;

            CLEAR (buf);

            buf.type        = V4L2_BUF_TYPE_VIDEO_CAPTURE;
            buf.memory      = V4L2_MEMORY_MMAP;
            buf.index       = n_buffers;

            if (-1 == xioctl (fd, VIDIOC_QUERYBUF, &buf))
                    errno_exit ("VIDIOC_QUERYBUF");

            buffers[n_buffers].length = buf.length;
            buffers[n_buffers].start =
                    mmap (NULL /* start anywhere */,
                          buf.length,
                          PROT_READ | PROT_WRITE /* required */,
                          MAP_SHARED /* recommended */,
                          fd, buf.m.offset);

            if (MAP_FAILED == buffers[n_buffers].start)
                    errno_exit ("mmap");
    }
}

static void init_device(void)
{
	struct v4l2_capability cap;
    struct v4l2_cropcap cropcap;
    struct v4l2_crop crop;
    struct v4l2_format fmt;
    unsigned int min;
    if (-1 == xioctl (fd, VIDIOC_QUERYCAP, &cap)) {
            if (EINVAL == errno) {
                    fprintf (stderr, "%s is no V4L2 device\n",
                             dev_name);
                    exit (EXIT_FAILURE);
            } else {
                    errno_exit ("VIDIOC_QUERYCAP");
            }
    }

    if (!(cap.capabilities & V4L2_CAP_VIDEO_CAPTURE)) {
            fprintf (stderr, "%s is no video capture device\n",
                     dev_name);
            exit (EXIT_FAILURE);
    }

    if (!(cap.capabilities & V4L2_CAP_STREAMING)) {
			fprintf (stderr, "%s does not support streaming i/o\n",
				 dev_name);
			exit (EXIT_FAILURE);
	}

	CLEAR (cropcap);

    cropcap.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

    if (0 == xioctl (fd, VIDIOC_CROPCAP, &cropcap)) {
            crop.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
            crop.c = cropcap.defrect; /* reset to default */

            if (-1 == xioctl (fd, VIDIOC_S_CROP, &crop)) {
                    switch (errno) {
                    case EINVAL:
                            /* Cropping not supported. */
                            break;
                    default:
                            /* Errors ignored. */
                            break;
                    }
            }
    } else {	
            /* Errors ignored. */
    }


    CLEAR (fmt);

    fmt.type                = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    fmt.fmt.pix.width       = WIDTH; 
    fmt.fmt.pix.height      = HEIGHT;
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
    fmt.fmt.pix.field       = V4L2_FIELD_INTERLACED;

    if (-1 == xioctl (fd, VIDIOC_S_FMT, &fmt))
            errno_exit ("VIDIOC_S_FMT");

    /* Note VIDIOC_S_FMT may change width and height. */

	/* Buggy driver paranoia. */
	min = fmt.fmt.pix.width * 2;
	if (fmt.fmt.pix.bytesperline < min)
		fmt.fmt.pix.bytesperline = min;
	min = fmt.fmt.pix.bytesperline * fmt.fmt.pix.height;
	if (fmt.fmt.pix.sizeimage < min)
		fmt.fmt.pix.sizeimage = min;

	init_mmap();
}

static void close_device(void)
{
    if (-1 == close (fd))
        errno_exit ("close");

    fd = -1;
}

static void open_device(void)
{
    struct stat st; 

    if (-1 == stat (dev_name, &st)) {
            fprintf (stderr, "Cannot identify '%s': %d, %s\n",
                     dev_name, errno, strerror (errno));
            exit (EXIT_FAILURE);
    }

    if (!S_ISCHR (st.st_mode)) {
            fprintf (stderr, "%s is no device\n", dev_name);
            exit (EXIT_FAILURE);
    }

    fd = open (dev_name, O_RDWR /* required */ | O_NONBLOCK, 0);

    if (-1 == fd) {
            fprintf (stderr, "Cannot open '%s': %d, %s\n",
                     dev_name, errno, strerror (errno));
            exit (EXIT_FAILURE);
    }
}

int main(int argc, char **argv)
{
	dev_name = "/dev/video0";

	fp = fopen("raw.yuv", "w+");
	if(0 == fp){
		printf("Error: failed to open file\n");
		exit(EXIT_FAILURE);
	}
	initialize_SDL();

	screen = SDL_CreateWindow("Disp", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED, screen_w, screen_h, SDL_WINDOW_OPENGL | SDL_WINDOW_RESIZABLE);

	if(!screen){
		fprintf(stderr, "SDL: cannot create window %s\n", SDL_GetError());
		return -1;
	}

	rect.x = 0;
	rect.y = 0;
	rect.w = screen_w;
	rect.h = screen_h;
	renderer = SDL_CreateRenderer(screen, -1, 0);
	texture = SDL_CreateTexture(renderer, SDL_PIXELFORMAT_IYUV, SDL_TEXTUREACCESS_STREAMING, pix_w, pix_h);

    open_device ();

    init_device ();

    start_capturing ();

    mainloop ();

    stop_capturing ();

    uninit_device ();

    close_device ();

    exit (EXIT_SUCCESS);

    return 0;
}

```




