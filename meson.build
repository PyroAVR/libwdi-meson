# Automate installation of USB device drivers in Windows
project(
    'libwdi',
    'c',
    version: '1.4.1',
)

windows = import('windows')

build_compiler = meson.get_compiler('c', native: true)
host_compiler = meson.get_compiler('c', native: false)
host_is_w64 = (host_compiler.get_define('__WIN64__') == '1')

libwdi_inc = include_directories(['libwdi'])


## Create config.h, autotools header
conf_data = configuration_data()
conf_data.set10('logging', get_option('logging'))
# these two flags appear to be redundant
conf_data.set10('debug_logging', get_option('debug_logging'))

foreach header : ['dlfcn', 'inttypes', 'stdint', 'stdio', 'stdlib', 'strings', 'string', 'sys_stat', 'sys_types', 'unistd']
    conf_data.set10(header, host_compiler.has_header(header + '.h'))
endforeach

conf_data.set_quoted('package_name', 'libwdi')
# TODO this version is wrong, but it is what is packaged at tag v1.4.1
conf_data.set_quoted('version', '1.3.1')
conf_data.set('wdf_ver', get_option('wdf_ver'))
# mesondefine will def/undef these correctly
if host_is_w64
    conf_data.set('OPT_M64', true)
else
    conf_data.set('OPT_M32', true)
endif


embed_driver_options = {
    'libusbk': get_option('libusbk_path'),
    'libusb0': get_option('libusb0_path'),
    'wdk': get_option('winusb_path'),
    'coinstaller': get_option('wdk_x64_path'),
    'user': get_option('user_path'),
}

foreach def, opt : embed_driver_options
    if opt != ''
        conf_data.set_quoted((def + '_dir').to_upper(), opt)
    endif
endforeach


config_header = configure_file(
    input: 'config.h.in.meson',
    output: 'config.h',
    configuration: conf_data
)
config_header_dep = declare_dependency(sources: config_header)

## Installer

# Installer runs on the host machine, but is required to build the embedder,
# since it is part of the embedded contents.
installer = executable('installer_' + (host_is_w64 ? 'x64':'x32'),
    files('libwdi/installer.c'),
    include_directories: libwdi_inc,
    link_args: [
        '-lsetupapi',
        '-lnewdev',
        '-lole32',
    ],
    dependencies: [config_header_dep],
)

## Embedder

# The embedder needs to run on the build machine: this program creates a header
# file that is the binary contents of the driver to install.
embedder_options = [
    '-DSRCDIR="' + meson.current_source_dir() / 'libwdi"',
    '-DSOLUTIONDIR="' + meson.current_build_dir() + '"',
]

embedder = executable('embedder',
    files(
        'libwdi/embedder.c',
    ),
    include_directories: libwdi_inc,
    c_args: embedder_options,
    dependencies: [config_header_dep],
    native: true,
)

## libwdi

# Create the header file that will contain the drivers + driver installer
embedded_h = custom_target(
    'embedded.h',
    output: 'embedded.h',
    # the argument to embedder is just a name
    command: [embedder, '@OUTPUT@'],
    depends: [installer]
)

# windows resource file defines exported symbols of libwdi
libwdi_winres = windows.compile_resources(files('libwdi/libwdi.rc'))

libwdi = library('wdi',
    [
        files(
            'libwdi/libwdi.c',
            'libwdi/logging.c',
            'libwdi/tokenizer.c',
            'libwdi/libwdi_dlg.c',
            'libwdi/pki.c',
            'libwdi/vid_data.c',
        ),
        embedded_h,
        libwdi_winres
    ],
    link_args: [
        '-lsetupapi',
        '-lole32',
        '-lntdll',
    ],
    dependencies: [config_header_dep],
    include_directories: libwdi_inc
)


## Examples

examples_inc = include_directories('examples')

## Zadig

# Graphical frontend for libwdi

zadig_winres = windows.compile_resources(files('examples/zadig.rc'))

zadig = executable('zadig',
    [
        files(
            'examples/zadig.c',
            'examples/zadig_net.c',
            'examples/zadig_parser.c',
            'examples/zadig_stdlg.c',
            'examples/profile.c',
        ),
        zadig_winres,
    ],
    link_args: [
        '-lwininet',
        '-luuid',
        '-lcomctl32',
    ],
    include_directories: [libwdi_inc, examples_inc],
    link_with: libwdi,
    gui_app: true,
)

## libwdi-simple

# console based frontend for libwdi

libwdi_simple_winres = windows.compile_resources(files('examples/wdi-simple.rc'))

libwdi_simple = executable('libwdi_simple',
    [
        files(
            'examples/wdi-simple.c',
        )
    ],
    include_directories: [libwdi_inc, examples_inc],
    link_with: libwdi,
    gui_app: false
)
